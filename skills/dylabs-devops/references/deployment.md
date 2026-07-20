# Deployment workflow

## 신규 또는 이전 배포

### 1. 앱별 배치 결정

먼저 현재 Application/source/workload/route/workflow를 조사한다. 다음은 힌트일 뿐 정책이 아니다.

- EKS: AWS controller나 AWS egress/allowlist, ARM64, EKS-local dependency가 필요한 workload
- OnPrem: amd64, internal/QA, Tunnel로 제공 가능한 public app, 비용 절감 대상
- GPU: 실제 target node GPU와 scheduling을 확인
- 외부 allowlist: source IP와 egress 경로를 전환 전에 확인

placement 변경은 별도 topology 결정으로 기록한다. “QA라서 OnPrem”, “prod라서 EKS” 같은 blanket rule을 쓰지 않는다.

### 2. Registry와 image

신규 DYLabs 앱은 GHCR을 기본으로 한다. DYLabs ECR을 새로 만들지 않는다. AWS native service가 GHCR을 지원하지 않는 등 검증된 예외가 있으면 Terraform ECR root와 consumer를 함께 설계한다.

- target architecture를 workflow와 target node에서 확인한다.
- tag가 source SHA인지, `qa-<sha>`, `onprem-<sha>` 등 prefix가 있는지 확인한다.
- multi-component 서비스는 source SHA 일치와 architecture별 digest 차이를 구분한다.
- private image는 pull Secret 선언뿐 아니라 namespace Secret Ready와 새 노드 fresh pull을 확인한다.

### 3. values

활성 Application이 실제 읽는 파일을 수정한다. 디렉터리명만 보고 `prod`, `qa`, `onprem`, `onprem-prod` 중 하나를 선택하지 않는다.

공통 점검:

- `global`, `app`, component/name
- `image.repository`, `image.tag` 또는 digest, pull policy, `imagePullSecrets`
- replicas/HPA/KEDA/Rollout 소유권
- requests와 memory limit, probe, service
- `externalSecret`, service account, RBAC
- ingress/Tunnel exposure
- architecture selector, affinity, toleration

현재 공통 chart는 `backend-app`, `cronjob-app`, `redirect` 세 개다. 과거 `frontend-app` 문서를 복사하지 말고 chart의 실제 values schema와 기존 active app을 읽는다.

OnPrem으로 옮길 때 EKS ARM/Karpenter selector를 제거하는 경우가 많지만 무조건 제거하지 않는다. live target node와 chart merge 결과를 확인한다.

### 4. Application

Application은 `k8s-manifests`에만 둔다.

- OnPrem Argo control plane의 apps root 아래에서 현재 패턴을 따른다.
- destination server/name과 namespace를 명시적으로 확인한다.
- Helm multi-source면 chart source와 `$values` ref/revision/path를 모두 확인한다.
- source revision을 고정해야 하는 migration/canary가 있는지 확인한다.
- parent/root App의 실제 registration 방식에 맞춘다.

수동 `kubectl apply`로 App CR을 만들지 않는다.

### 5. CI

기본은 GitHub-hosted runner다. target network 접근이 필요한 job은 Tailscale 또는 현재 승인된 runner 경로를 사용한다.

- EKS ARC label과 `builder` NodePool을 재도입하지 않는다.
- 과거 onprem runner label도 현재 등록·용량·보안 승인을 확인하지 않고 복사하지 않는다.
- build architecture와 target node architecture를 일치시킨다.
- workflow가 활성 values path를 갱신하는지 확인한다.
- monorepo path filter 때문에 workflow-only commit이 실행되지 않는지 확인한다.
- Vault는 `https://vault.pri.sbalyd.com`을 사용한다. 폐기된 EKS in-cluster URL을 재도입하지 않는다.
- migration job은 대상 DB, network, migration ordering, app rollout과의 결합을 별도로 확인한다.

특수 worker/KEDA 서비스는 일반 Helm rollout이 아닐 수 있다. `meloming-chat-worker`처럼 workflow가 KEDA pause/image/unpause를 직접 제어하는 예외는 현재 workflow와 운영 메모리를 읽고 그대로 검증한다. 이를 다른 서비스의 기본 패턴으로 복사하지 않는다.

### 6. Secret

Vault KV v2 CLI path와 ExternalSecret `remoteRef` 표현을 구분한다. 대체로 CLI는 mount를 제외한 `<env>/<namespace>/<service>` 형태를 쓰고 policy는 `secret/data/<path>`를 본다. 실제 manifest를 확인한다.

- secret value를 커맨드 인자에 직접 넣지 않는다.
- 기존 policy를 read-modify-write할 때 다른 path를 보존한다.
- policy 변경 뒤 AppRole token TTL 때문에 반영 지연이 있을 수 있다.
- ExternalSecret Ready만 보지 말고 target Secret과 workload event를 확인한다.

### 7. Exposure

Private:

- Route53 `*.pri.sbalyd.com` source와 Tailnet endpoint 확인
- Ingress/controller/certificate와 실제 HTTP 경로 확인

Public OnPrem:

- ClusterIP service
- cloudflared ingress `hostname -> service.namespace.svc`
- Cloudflare DNS CNAME/proxy
- cloudflared config reload/rollout 메커니즘 확인
- catch-all rule보다 앞에 hostname이 있는지 확인

Public EKS:

- ALB group/listener/certificate/external-dns와 backend readiness 확인

DNS 전환과 workload 배포를 분리해 양쪽을 검증하고 rollback 기준을 준비한다.

## 배포 완료 게이트

```bash
gh run list --branch <branch> --limit 5
kubectl --context dylabs-onprem -n argocd get application <app>
kubectl --context <destination> -n <namespace> get pods -l <tracking-label> -o wide
kubectl --context <destination> -n <namespace> get pods -l <tracking-label> \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,IMAGE:.spec.containers[*].image,RESTARTS:.status.containerStatuses[*].restartCount'
```

Deployment template image는 desired state일 뿐 rollout 완료 신호가 아니다. 다음을 만족해야 한다.

- old image Pod 0
- new image Ready Pod가 expected replica 이상
- rollout 중 Pending이면 event로 image pull과 scheduling failure를 구분
- 실제 Pod image/digest가 CI artifact와 일치
- API/DB/Redis/Vault 등 핵심 의존성 오류가 없음
- public/private 사용자 경로가 실제로 응답

폴링 query의 stderr를 무조건 버리지 않는다. 빈 결과, jsonpath 오류, tag parsing 오류를 timeout으로 오인하지 않는다.

## Prisma와 DB migration

Prisma schema/migrate 작업은 `prisma-migration` skill도 사용한다. production DB 직접 DDL/DML을 실행하지 않는다. 배포 workflow의 migration job이 app rollout 전후 어디에 위치하는지와 실패 시 rollback 가능성을 확인한다.

