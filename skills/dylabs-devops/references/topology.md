# Topology and ownership

이 문서는 2026-07-21 KST에 코드·Codex 작업 기록·라이브 클러스터를 대조한 기준이다. 숫자와 개별 앱 상태는 변하므로 정책으로 고정하지 않는다.

## 리포 소유권

| 리포 | 소유 범위 |
|---|---|
| `terraform-main` | AWS IaC, DNS/IAM/S3/RDS/ElastiCache/EKS와 AWS-side OnPrem dependency |
| `onprem` | KoreaIDC 호스트와 k3s 플랫폼 Ansible, Vault 운영 문서 |
| `k8s-manifests` | 모든 Argo CD Application과 cluster/platform manifest |
| `deploy-gitops` | 앱별 Helm values와 raw workload manifest. Application을 두지 않는다. |
| `devops-helm` | 공통 chart: `backend-app`, `cronjob-app`, `redirect` |
| `monitoring-gitops` | rules, dashboards, Alertmanager config, exporter manifest |

리포 문서보다 live가 최신일 수 있지만, live만 보고 out-of-band 변경하지 않는다. 불일치는 source와 owner를 찾아 정리한다.

## Argo CD 제어면

- OnPrem `argocd` namespace가 유일한 active control plane이다.
- OnPrem Application이 OnPrem 또는 EKS destination을 가질 수 있다. 앱 이름에 `-eks`가 붙는 패턴이 있지만 이름만으로 확정하지 않는다.
- EKS의 Application CR과 Argo CD Pod/control plane은 0이다.
- OnPrem root app은 apps directory를 관리하며 top-level Application YAML을 등록한다. 정확한 root source와 recurse 설정을 live App에서 확인한다.
- EKS 대상 Application 선언도 현재는 `clusters/dylabs-onprem-k3s/apps/` 아래에 존재할 수 있다. 과거 EKS kustomization만 보고 소유권을 단정하지 않는다.

조회 출발점:

```bash
kubectl --context dylabs-onprem -n argocd get applications.argoproj.io
kubectl --context dylabs-onprem -n argocd get application <app> -o yaml
kubectl --context dylabs-onprem -n argocd get deploy,statefulset
kubectl --context dylabs-prod-eks-main -n argocd get applications.argoproj.io
```

### Application 삭제 안전

삭제 전 rendered manifest와 live `.status.resources`를 확인한다. 다음이 있으면 단일 commit의 cascading deletion을 중단한다.

- Namespace
- cluster-scoped resource
- shared Secret, routing, operator, CRD
- 다른 Application workload가 있는 namespace

Namespace 소유 helper App은 단계적으로 처리한다.

1. Namespace에 `Prune=false`를 선언하고 live 반영 확인
2. helper resource만 prune하고 Namespace가 남는지 확인
3. Application finalizer 제거를 선언·확인
4. 그 뒤 Application/source 제거
5. parent App, affected App, Namespace, workload, pull Secret, route 재검증

## 클러스터

| 영역 | 현재 기준 |
|---|---|
| EKS | `dylabs-prod-eks-main`, `ap-northeast-2`, 현재 worker는 ARM64 중심 |
| OnPrem k3s | context `dylabs-onprem`, 3-node amd64 |
| Argo CD | OnPrem HA control plane, EKS와 OnPrem을 함께 관리 |
| OnPrem KEDA | 실제 chat worker consumer가 있어 유지 |
| EKS KEDA | consumer 0 확인 후 퇴역 완료 |
| EKS ARC | controller/runner/CRD/namespace/builder NodePool 퇴역 완료 |

`dylabs-onprem-3`~`6` standalone 호스트는 별도 서버이며 자동으로 k3s worker가 아니다. inventory와 live node 목록을 함께 확인한다.

## Registry

- 기본: `ghcr.io/dylabs/{qa|prod}/<image>` 패턴. 실제 이름과 tag 규칙은 workflow와 live image에서 확인한다.
- EKS와 OnPrem의 active DYLabs 앱 Pod는 GHCR로 이관됐다.
- AWS-managed ECR system image와 DYLabs 앱 ECR을 구분한다.
- DYLabs ECR의 알려진 영구 예외는 Lambda image consumer가 있는 `qa/meloming-image-worker`, `prod/meloming-image-worker`다. 변경 전 AWS consumer를 다시 조회한다.
- dormant values, old ReplicaSet/Job, ECR refresh manifest는 active consumer가 아니다. 그렇다고 문자열만 보고 즉시 삭제하지 않는다.
- private GHCR workload는 namespace의 `ghcr-pull` Secret/ExternalSecret과 실제 fresh pull을 확인한다.

## Vault/ESO

- endpoint: `https://vault.pri.sbalyd.com`
- authoritative server: OnPrem Vault 3 replicas, integrated Raft
- OnPrem ESO: cluster-local active service와 `onprem-external-secrets` AppRole
- EKS ESO: production endpoint와 EKS 전용 AppRole
- snapshot uploader: EKS CronJob이 전용 read-only AppRole로 OnPrem endpoint snapshot을 S3/KMS/Object Lock에 저장
- EKS old Vault: StatefulSet `0/0`, PVC 보존. rollback은 clean target에 최신 snapshot restore 후 client 전환으로 설계한다.

AppRole, policy, endpoint는 역할별로 분리한다. 기존 EKS Vault와 OnPrem Vault를 동시에 writable로 열지 않는다.

## Routing

- private 표준은 `*.pri.sbalyd.com`이며 Route53/Tailnet 경로를 사용한다.
- `*.int.sbalyd.com`은 제거된 역사적 namespace다. 새 값에 재도입하지 말고 live/source 0 여부를 확인한다.
- public OnPrem 앱은 단일 `dylabs-onprem` Cloudflare Tunnel의 ingress rule로 여러 service를 연결한다.
- EKS public 앱은 ALB/Ingress/ACM/external-dns 패턴을 쓸 수 있다.
- Cloudflare Tunnel과 EKS ALB 중 어느 경로인지 hostname별 DNS, tunnel config, Ingress, backend service를 모두 확인한다.

Vercel 흔적이나 응답 헤더만으로 DYLabs/Meloming 배포 위치를 판단하지 않는다. 앱별 live chain에서 Vercel이 실제 active deployment인지 확인한다.

