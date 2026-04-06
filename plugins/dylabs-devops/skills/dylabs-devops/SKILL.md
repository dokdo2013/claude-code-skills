---
name: dylabs-devops
description: "Use when deploying new services, configuring monitoring, managing Terraform infrastructure, debugging Kubernetes issues, or any DevOps/infrastructure task for dylabs. Triggers on: 배포, 배포 설정, 모니터링, 인프라, Terraform, EKS, ArgoCD, Helm, 신규 서비스 배포, deploy, infra, monitoring, alert, dashboard"
---

# dylabs DevOps Runbook

dylabs 인프라 배포/모니터링/운영 프로시저.

## ⛔ 직접 CLI가 있으면 우회하지 마라 (CRITICAL)

**아래 CLI가 이미 설치되어 있다. 직접 CLI로 해결할 수 있는 작업을 우회 방법으로 처리하면 안 된다.**

금지 패턴 예시:
- `vault` CLI 있는데 → `kubectl port-forward` 해서 Vault API 호출 ❌
- `mysql` CLI 있는데 → Node.js + Prisma로 쿼리 실행 ❌
- `helm` CLI 있는데 → `kubectl apply`로 수동 매니페스트 적용 ❌
- `aws` CLI 있는데 → curl로 AWS API 직접 호출 ❌
- `gh` CLI 있는데 → curl로 GitHub API 호출 ❌

**직접 CLI가 있으면 그걸 써라. 간접 경로로 돌아가지 마라.**

### 인프라 & 클라우드

| CLI | 경로 | 버전 | 용도 |
|-----|------|------|------|
| `terraform` | `/opt/homebrew/bin/terraform` | v1.14.2 | IaC — `terraform-main` 리포 전용. tfenv으로 버전 관리 |
| `aws` | `/usr/local/bin/aws` | 2.32.3 | AWS 리소스 조회/관리 |
| `session-manager-plugin` | `/opt/homebrew/bin/session-manager-plugin` | 1.2.707.0 | AWS SSM 세션 (EC2 접속, 포트 포워딩) |
| `kubectl` | `/opt/homebrew/bin/kubectl` | v1.34.2 | K8s 클러스터 관리. kustomize v5.7.1 내장 |
| `helm` | `/opt/homebrew/bin/helm` | v4.0.0 | Helm chart 관리 |
| `vault` | `/opt/homebrew/bin/vault` | v1.21.1 | HashiCorp Vault 시크릿 관리 |

### 컨테이너

| CLI | 경로 | 버전 | 용도 |
|-----|------|------|------|
| `docker` | `/opt/homebrew/bin/docker.lima` | 29.1.3 | 컨테이너 빌드/실행 (Lima 기반, Docker Desktop 아님) |
| `docker compose` | docker plugin | v5.0.0 | 로컬 멀티컨테이너 (`docker compose up`) |

### CI/CD & Git

| CLI | 경로 | 버전 | 용도 |
|-----|------|------|------|
| `gh` | `/opt/homebrew/bin/gh` | 2.83.1 | GitHub PR/이슈/워크플로우/시크릿 관리 |

### DB 접속 (⛔ SELECT ONLY — 변경 쿼리 절대 금지)

| CLI | 경로 | 버전 | 용도 |
|-----|------|------|------|
| `mysql` | `/opt/homebrew/bin/mysql` | 8.0.44 | MySQL/Aurora 접속. **읽기 전용** |
| `redis-cli` | `/opt/homebrew/bin/redis-cli` | 8.4.0 | Redis 접속. **읽기 전용** |

### 유틸리티

| CLI | 경로 | 버전 | 용도 |
|-----|------|------|------|
| `jq` | `/usr/bin/jq` | 1.7.1 | JSON 파싱/필터링 |
| `rg` | `/opt/homebrew/bin/rg` | 15.1.0 | ripgrep — 코드/설정 파일 검색 |
| `envsubst` | `/opt/homebrew/bin/envsubst` | GNU 1.0 | 환경변수 치환 (deploy 템플릿 등) |
| `curl` | `/usr/bin/curl` | 8.7.1 | HTTP 요청/API 호출 |

### 미설치 — 사용 금지

`argocd` (CLI), `stern`, `kubectx`, `kubens`, `kustomize` (standalone), `yq`, `sops`, `skopeo`, `crane`, `istioctl`, `kedactl`, `pulumi`

> ArgoCD는 CLI 없이 `kubectl get app -n argocd` 로 관리. kustomize는 `kubectl` 내장 버전 사용.

---

## 아키텍처 개요

### 리포 구조

| 리포 | 역할 |
|------|------|
| `terraform-main` | IaC (Terraform) — AWS 인프라 전체 관리 |
| `k8s-manifests` | ArgoCD Application 정의 + kustomization |
| `deploy-gitops` | 환경별 Helm values (`apps/{service}/{env}/values.yaml`) |
| `devops-helm` | 공통 Helm charts (`backend-app`, `frontend-app` 등) |
| `monitoring-gitops` | PrometheusRule, Grafana Dashboard, Alertmanager 설정 |

### 배포 플로우

```
git push (service repo)
  → GitHub Actions CI
    → Test
    → Docker build (ARM64) & push to ECR
    → DB migration (if applicable)
    → update image tag in deploy-gitops
    → ArgoCD detects change → auto sync → K8s rollout
```

### 주요 인프라

| 구성 | 값 |
|------|-----|
| EKS Cluster | `dylabs-prod-eks-main` |
| Region | `ap-northeast-2` (서울) |
| Node Manager | Karpenter (ARM64, Spot 선호) |
| IaC | Terraform (S3 remote state + DynamoDB lock) |
| Secrets | Vault + External Secrets Operator |
| CI Runner | GitHub ARC (K8s pod, ARM 빌드/클러스터 내부 접근 시) + ubuntu-latest |

### 프론트엔드 배포

일부 프론트엔드 앱은 **Vercel**을 사용 (git push → Vercel 자동 배포).
나머지 프론트엔드는 EKS + ArgoCD 배포.

---

## 1. 신규 서비스 배포 설정 (체크리스트)

새 서비스를 처음 배포할 때 따라야 하는 절차.

### Step 1: Terraform으로 ECR 리포지토리 생성

**ECR은 반드시 Terraform으로만 관리.** 수동 AWS CLI 생성 금지.

`terraform-main` 리포에서 ECR 리소스 추가:

```hcl
module "ecr_{service}" {
  source   = "./modules/ecr"
  name     = "{service-name}"
  env      = var.environment
  tags     = local.common_tags
}
```

```bash
cd terraform-main
terraform plan  -var-file=env/prod.tfvars -out=tfplan
# ⛔ plan 결과를 사용자에게 보여주고 승인 후 apply
terraform apply tfplan
```

### Step 2: deploy-gitops에 values.yaml 생성

`deploy-gitops/apps/{service}/{env}/values.yaml` — 환경별로 필요한 데이터:

**공통 필드:**
- `global.environment`, `global.region`
- `app.name`, `app.component`
- `image.repository` (ECR URI), `image.tag`
- `replicaCount`, `resources` (requests/limits)

**필요에 따라 설정:**
- `ingress` — ALB 호스트, 인증서, path 라우팅
- `healthCheck` — liveness/readiness probe
- `externalSecret` — Vault 경로, 타겟 시크릿명
- `env` — 환경변수 목록
- `metrics.serviceMonitor` — Prometheus 메트릭 수집

> **참고**: 기존 서비스의 values.yaml을 참고하여 작성. `devops-helm/charts/backend-app/values.yaml`에서 전체 스키마 확인 가능.

### Step 3: k8s-manifests에 ArgoCD Application 등록

`k8s-manifests/clusters/dylabs-prod-eks-main/apps/` 에 추가:

**필요한 데이터:**
- Application name (= 서비스명)
- Helm chart 경로 (`devops-helm/charts/{chart-name}`)
- values 파일 참조 (`deploy-gitops/apps/{service}/{env}/values.yaml`)
- 대상 namespace
- Rollout 사용 시 `ignoreDifferences` 설정

> **참고**: 기존 App YAML을 복사하고 서비스명/namespace/chart 경로만 수정.

등록 후 `kustomization.yaml`의 `resources:`에도 추가.

### Step 4: CI/CD Workflow 추가

서비스 리포에 `.github/workflows/` 에 QA/Prod 워크플로우 생성.

**필수 Org Secrets** (레포 시크릿 아님, dylabs Org에 이미 설정됨):
| Secret | 용도 |
|--------|------|
| `DEPLOY_GITOPS_TOKEN` | deploy-gitops 리포 접근용 토큰 |
| `VAULT_ROLE_ID` | Vault 인증 Role ID |
| `VAULT_SECRET_ID` | Vault 인증 Secret ID |
| `SLACK_WEBHOOK_URL` | 배포 알림용 Slack Webhook |

**Runner 선택 기준:**
| Job | Runner | 이유 |
|-----|--------|------|
| Test | `ubuntu-latest` | 클러스터 접근 불필요 |
| Docker build (ARM) | `arc-runner` | ARM64 빌드 필요 |
| DB migration | `arc-runner` | Vault/클러스터 접근 필요 |
| deploy-gitops update | `ubuntu-latest` | git push만 하면 됨 |

**워크플로우 구조:**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # 프로젝트에 맞는 테스트 실행
      - run: {test-command}

  build-and-push:
    needs: test
    runs-on: arc-runner
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::{account}:role/{irsa-role-name}
          aws-region: ap-northeast-2
      - name: Build and push (ARM64)
        uses: aws-actions/amazon-ecr-login@v2
        # ... docker buildx build --platform linux/arm64

  migrate:  # Prisma 등 DB migration 필요 시
    needs: build-and-push
    runs-on: arc-runner
    steps:
      # Vault에서 DB 접속 정보 가져오기
      # migration 실행

  deploy:
    needs: [build-and-push, migrate]  # migrate가 없으면 build-and-push만
    runs-on: ubuntu-latest
    if: always() && needs.build-and-push.result == 'success' && (needs.migrate.result == 'success' || needs.migrate.result == 'skipped')
    steps:
      # deploy-gitops clone → sed image tag → push
```

**QA와 Prod 모두 동일한 CI 파이프라인을 타야 함.** QA만 수동 이미지 태그 업데이트 금지.

### Step 5: Vault에 시크릿 등록

Vault KV v2 경로: `secret/data/{env}/{namespace}/{service-name}`
CLI에서는 `secret/{env}/{namespace}/{service-name}` 로 접근 (Vault가 자동으로 `data/` 경로에 매핑).

```bash
# 값은 stdin이나 파일로 전달하여 shell history 노출 방지
kubectl exec -n vault vault-0 -- vault kv put secret/{env}/{namespace}/{service-name} -
# 또는 Vault UI / External Secrets Operator 사용
```

### Step 6: 배포 확인

```bash
# ArgoCD sync 상태
kubectl get application -n argocd {service-name}

# Pod 상태
kubectl get pods -n {namespace} -l app.kubernetes.io/name={service-name}

# 로그
kubectl logs -n {namespace} -l app.kubernetes.io/name={service-name} --tail=50

# Ingress
kubectl get ingress -n {namespace}
```

---

## 2. 일상적 배포 (레퍼런스)

### 일반 배포 (CI 자동)

`main` 브랜치 push → CI 전체 파이프라인 자동 실행 (test → build → deploy).

### 수동 배포 (긴급 핫픽스 등)

deploy-gitops에서 수동으로 이미지 태그 업데이트 후 push:

```bash
# values.yaml의 tag 필드를 원하는 커밋 SHA로 변경 후 커밋/푸시
# ArgoCD가 자동 감지하여 sync
```

### Pod 재시작 (ArgoCD sync와 별개)

ArgoCD는 git 기반이므로 **Git의 values.yaml을 변경하는 것이 정석**. 아래는 긴급 Pod 재시작만 수행:

```bash
# Pod 재시작 (ArgoCD sync가 아님 — ArgoCD가 drift를 감지하면 되돌릴 수 있음)
kubectl rollout restart deployment -n {namespace} {deployment-name}
```

### 배포 상태 확인

```bash
kubectl get app -n argocd {service-name} -o jsonpath='{.status.health.status}'
kubectl argo rollouts get rollout -n {namespace} {rollout-name}
kubectl get events -n {namespace} --sort-by='.lastTimestamp' | tail -20
```

### Worker 서비스 (KEDA 관리)

ArgoCD 관리가 아닌 Worker는 `deploy-worker.yml`로 직접 배포:

```bash
# 1. KEDA pause
kubectl annotate scaledobject -n {namespace} {name} \
  autoscaling.keda.sh/paused=true --overwrite

# 2. 이미지 업데이트
kubectl set image deployment -n {namespace} {deployment} {container}={image}:{tag}

# 3. KEDA unpause
kubectl annotate scaledobject -n {namespace} {name} \
  autoscaling.keda.sh/paused- --overwrite
```

---

## 3. 모니터링 추가 (체크리스트)

### Step 1: PrometheusRule 추가

`monitoring-gitops/base/rules/{alert-name}.yaml`

**필요한 데이터:**
- Alert name, PromQL expression, 지속 시간 (`for`)
- Severity (`warning` / `critical`)
- Summary/Description annotation

### Step 2: Grafana 대시보드 추가

`monitoring-gitops/base/dashboards/{category}/{name}.yaml`

ConfigMap + `grafana_dashboard: "1"` 라벨로 Grafana가 자동 인식.

카테고리: `infrastructure/`, `application/`, `kubernetes/`

### Step 3: Alertmanager 라우팅

| Severity | Slack Channel | incident.io |
|----------|--------------|-------------|
| critical | #alerts-critical | Yes |
| warning | #alerts-warning | Yes |

### Step 4: ServiceMonitor (필요 시)

values.yaml에서 `metrics.serviceMonitor.enabled: true` 설정, 또는 직접 ServiceMonitor 리소스 생성.

### Step 5: 배포 및 확인

```bash
# monitoring-gitops push → ArgoCD 자동 sync
kubectl get prometheusrules -n monitoring
kubectl get configmaps -n monitoring -l grafana_dashboard=1
kubectl get alertmanagerconfigs -n monitoring
```

---

## 4. Terraform 인프라 작업 (레퍼런스)

### 작업 절차

```bash
cd terraform-main
git pull origin main

# 변경 계획 확인
terraform plan -var-file=env/{env}.tfvars -out=tfplan

# ⛔ plan 결과를 사용자에게 보여주고 승인 후 apply
terraform apply tfplan

# 커밋
git add . && git commit -m "infra: {변경 내용}" && git push origin main
```

### 네이밍 컨벤션

- 패턴: `{project}-{env}-{resource}-{identifier}`
- 태그: `Environment`, `Project`, `ManagedBy`, `ResourceType`, `Name`

### 주의사항

- **`terraform apply` 전 반드시 plan 결과를 사용자에게 보여줄 것**
- **Prod 리소스 삭제/수정은 사용자 확인 필수**
- ECR, RDS, ElastiCache, VPC, SG, IAM 등 모든 AWS 리소스를 Terraform으로만 관리

---

## 5. 장애 대응 (레퍼런스)

### 기본 디버그 명령어

```bash
# Pod 상태
kubectl get pods -n {namespace}
kubectl describe pod -n {namespace} {pod-name}

# 로그
kubectl logs -n {namespace} {pod-name} --tail=200
kubectl logs -n {namespace} {pod-name} --previous

# 이벤트
kubectl get events -n {namespace} --sort-by='.lastTimestamp' | tail -30

# 리소스 사용량
kubectl top pods -n {namespace}
kubectl top nodes
```

### 장애 패턴별 대응

| 증상 | 확인 명령어 | 일반적 원인 |
|------|------------|------------|
| CrashLooping | `kubectl logs {pod} --previous` | 앱 에러, config 누락 |
| OOMKilled | `kubectl describe pod {pod} \| grep -A5 Last` | 메모리 부족 → values에서 limits 증가 |
| Replicas Mismatch | `kubectl describe deployment {name}` | 스케줄링 실패, 리소스 부족 |
| ArgoCD Sync 실패 | `kubectl get app -n argocd {name} -o yaml` | values 에러, chart 호환성 |
| Vault 시크릿 오류 | `kubectl get externalsecret -n {ns}` | Vault 경로/권한 문제 |
