---
name: dylabs-devops
description: "Operate and change DYLabs infrastructure safely: Kubernetes/EKS/k3s, Argo CD/GitOps, GitHub Actions and container registries, Terraform/AWS, Vault/External Secrets, Cloudflare/Route53/Tailscale networking, monitoring, capacity, migrations, deployments, and incidents. Use for 배포, 배포 설정, 배포 확인, 롤아웃, 인프라, 온프렘/onprem, EKS, k3s, Kubernetes, ArgoCD, GitOps, Helm, GHCR/ECR, Terraform, AWS, Vault, ESO, DNS, Cloudflare Tunnel, Tailscale, 모니터링, 알림, Grafana, Prometheus, Loki, 장애 대응, 비용·용량 최적화, 신규 서비스 배포, deploy, rollout, infra, monitoring, alert, dashboard, SRE, or DevOps work in DYLabs/Meloming systems."
---

# DYLabs DevOps

현재 코드·라이브 상태·운영 규칙을 함께 대조해 DYLabs 인프라를 진단하고 변경한다. 환경명이나 과거 메모리만으로 배치 위치와 배포 경로를 추론하지 않는다.

## 1. 시작 게이트

작업 전에 다음을 순서대로 수행한다.

1. `/Users/hyeonwoo/DEV/AGENTS.md`와 `/Users/hyeonwoo/DEV/CLAUDE.md`를 읽는다.
2. 수정하거나 운영 명령을 실행할 각 리포의 `AGENTS.md`와 `CLAUDE.md`를 읽는다. 없는 파일은 첫 업데이트에 명시한다.
3. Claude memory의 `MEMORY.md`, `critical-rules.md`, `MEMORY-TODO.md`, `DAILY-TASKS.md`와 작업에 관련된 최신 `project-*`, `reference-*`, `feedback-*`를 읽는다.
4. 리포 상태, 원격 브랜치, 다른 세션 변경을 확인한다. 다른 세션 자산은 삭제·rename·revert·덮어쓰기하지 않는다.
5. 진단 요청은 읽기 전용으로 끝낸다. 변경 요청만 실제 변경 권한으로 해석한다.
6. 사용자에게 첫 업데이트로 확인한 리포 규칙, 읽기 전용/변경 범위, 로컬 검증 금지를 알린다.

## 2. 절대 규칙

- 로컬 테스트·빌드·린트·타입체크·패키징·dev/watch 서버·실기기 빌드를 실행하지 않는다. Docker 이미지도 로컬에서 빌드하지 않는다. 원격 CI와 실제 배포 상태로 검증한다.
- Production DB에는 `SELECT`만 실행한다. QA DB 변경도 사용자 사전 승인을 받는다.
- `meloming-back` Controller/Service에 PostHog feature flag guard를 추가하지 않는다.
- 비밀값을 출력, Git 커밋, 대화 인용, shell history 노출하지 않는다. 존재·버전·해시 일치만 검증한다.
- OnPrem Kubernetes 변경은 Git → reconciler로만 수행한다. `kubectl apply/edit/patch/delete/rollout restart`, 수동 App CR, `helm upgrade`를 사용하지 않는다. P0 또는 사용자 명시 승인 예외도 즉시 소스와 정합성을 복구한다.
- Terraform `apply`, production 리소스 변경·삭제, 데이터 손실 가능 작업은 정확한 plan과 영향 범위를 제시하고 사용자의 명시 승인을 받은 뒤 수행한다.
- 임시방편으로 증상을 숨기지 않는다. P0 응급 완화는 임시 조치임을 밝히고 근본 해결을 별도 추적한다.
- 파일을 변경하면 사용자가 금지하지 않는 한 자신의 변경만 commit/push하고 원격 반영과 해당 CI/rollout을 확인한다.

실수로 금지된 로컬 검증 프로세스를 발견하면 즉시 중단하고 보고한다.

## 3. 작업별 reference

필요한 reference를 작업 전에 끝까지 읽는다.

- 앱 위치, Argo CD 소유권, 클러스터·레지스트리·리포 구조: [references/topology.md](references/topology.md)
- 신규 서비스, CI, values, Application, 배포·롤아웃 검증: [references/deployment.md](references/deployment.md)
- Terraform, Vault/ESO, DNS, Tunnel, Tailscale, OnPrem 플랫폼: [references/infrastructure.md](references/infrastructure.md)
- 모니터링, 용량, 장애 진단, 완료 판정: [references/operations.md](references/operations.md)

여러 영역이면 관련 reference를 모두 읽되, 작업과 무관한 파일은 로드하지 않는다.

## 4. 토폴로지 우선 원칙

배포·레지스트리·Application·namespace·destination 변경 전 아래 체인을 앱별로 확정한다.

1. live Argo CD Application의 source, values, revision, destination cluster, namespace
2. live controller와 Pod의 tracking ID, image/digest, replicas, rollout 상태
3. Ingress, Cloudflare Tunnel, Route53/Cloudflare DNS, private route 등 실제 트래픽 경로
4. 이미지를 발행하고 활성 GitOps path를 갱신하는 source workflow와 trigger branch
5. dormant values, 오래된 ReplicaSet/Job, 문서, registry repo는 마지막에 정리할 2차 근거

`qa`, `prod`, `onprem`, `onprem-prod`, 브랜치명, 디렉터리명, Argo CD가 실행되는 클러스터만으로 destination을 정하지 않는다. 동일 제어면이 두 클러스터를 관리할 수 있다.

현재 구조의 핵심은 다음과 같지만, 이는 2026-07-21 스냅샷이며 매 작업마다 재검증한다.

- Argo CD control plane은 OnPrem에 있고 EKS와 OnPrem destination을 함께 관리한다. EKS 내부 Argo CD control plane은 퇴역했다.
- DYLabs 애플리케이션 이미지의 기본 registry는 GHCR이다. ECR은 AWS native consumer 등 검증된 예외만 유지한다.
- OnPrem k3s는 3-node amd64 클러스터다. 별도 standalone 서버를 k3s 노드로 오인하지 않는다.
- authoritative Vault는 OnPrem 3-node Raft다. EKS Vault StatefulSet은 `0/0` cold source이며 단순 scale-up하지 않는다.
- EKS ARC runner와 builder NodePool은 퇴역했다. 새 workflow에 ARC label을 재도입하지 않는다.

## 5. 실행 원칙

### CLI

사용 전 `command -v`와 필요 시 `--version`으로 현재 설치 상태를 확인한다. 과거 경로·버전을 사실로 가정하지 않는다.

- 목적 전용 CLI가 있으면 그 CLI를 사용한다: `terraform`, `aws`, `kubectl`, `helm`, `vault`, `gh`, `mysql`, `redis-cli`.
- 직접 CLI가 없거나 인증이 없으면 임의 API 우회, port-forward, 앱 런타임을 이용한 대체 접근을 만들지 않는다.
- `argocd` CLI가 없으면 OnPrem context의 Application CR과 GitOps source로 확인한다.
- `docker`가 없거나 로컬 빌드가 금지되어도 설치·우회 빌드하지 않는다. CI를 사용한다.

### GitOps와 공유 작업

- push 전 원격 최신 상태와 shared-repo operation을 확인한다. 다른 세션이 같은 Application, values, Terraform state를 소유하면 직렬화한다.
- Application이 `Unknown`, `ComparisonError`, `Degraded`이거나 operation이 Running이면 공유 root 변경을 멈추고 원인을 확인한다.
- live patch로 끝내지 않고 source of truth를 수정한다.
- Application 삭제 전 `.status.resources`, finalizer, Namespace·cluster-scoped·shared resource 소유를 확인한다. Namespace 소유 앱은 한 번에 cascading delete하지 않는다.

### Terraform

- 정확한 root, backend config, workspace, state drift를 먼저 확인한다.
- 저장 plan은 다른 apply 후 stale하다. 승인 직전 최신 코드/state로 다시 plan한다.
- target은 공유 drift를 숨길 수 있으므로 exact resource 범위와 full-plan 차이를 함께 검토한다.
- plan 파일, `.terraform` 산출물, lock 임시 변경을 임의 커밋하지 않는다.

## 6. 완료 판정

GitHub Actions success, Argo CD `Synced/Healthy`, Deployment desired image 중 하나만으로 “배포 완료”라고 말하지 않는다.

모두 확인한다.

1. source commit과 remote branch
2. remote CI 성공과 실제 발행 image/digest
3. 활성 GitOps values가 올바른 tag/digest를 가리키는지
4. Application revision, sync, health, operation phase
5. 실제 Pod 목록에서 이전 이미지 Pod 0, 새 이미지 Ready Pod가 기대 replica 수만큼 존재하는지
6. 새 Pod의 image/digest, restart, event, 핵심 의존성 오류
7. 실제 사용자 경로의 DNS/Tunnel/Ingress/HTTP/API 동작

멀티세션 브랜치에서 후속 커밋이 배포됐으면 내 commit이 실행 image SHA의 조상인지 확인한다. tag prefix가 있으면 실제 image 문자열을 먼저 보고 파싱한다. 조회값이 비면 “진행 중”으로 숨기지 말고 query failure로 보고한다.

완료 보고에는 확인한 규칙과 근거, 변경 리포/파일, commit/push/CI/rollout/route 상태, 남은 위험, 로컬 검증을 하지 않은 이유를 포함한다.

