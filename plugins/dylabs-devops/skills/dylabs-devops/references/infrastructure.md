# Infrastructure operations

## Source of truth

| 영역 | 소스 | 적용 |
|---|---|---|
| AWS resource | `terraform-main`의 정확한 root | approved saved plan apply |
| OnPrem host/k3s platform | `onprem` Ansible roles/playbooks | Git push 후 승인된 reconciler/playbook |
| Application | `k8s-manifests` + `deploy-gitops` + `devops-helm` | OnPrem Argo CD auto-sync |
| Monitoring content | `monitoring-gitops`와 stack Application | Argo CD |

AWS console/CLI mutation, live Kubernetes mutation, node-local 수동 파일 수정으로 source를 우회하지 않는다.

## Terraform

작업 root를 먼저 찾는다. 알려진 예:

- `aws/ap-northeast-2/onprem-k3s`: private DNS와 OnPrem AWS dependency
- `aws/ap-northeast-2/tailscale-router`: VPC↔Tailnet routing
- `aws/ap-northeast-2/ecr`: DYLabs ECR 예외와 retirement
- `aws/ap-northeast-2/eks`, `eks-addons`, `rds`, `elasticache`, `s3`
- Cloudflare/DNS는 현재 실제 provider root를 검색해 확정

안전 절차:

1. 리포 규칙, branch, remote 최신 상태, 다른 세션 owner를 확인한다.
2. backend config와 workspace를 확인한다.
3. refresh/state 오류를 숨기지 않고 plan 전체를 읽는다.
4. exact intended delta와 unrelated drift를 분리한다.
5. plan을 사용자에게 제시하고 apply 승인을 받는다.
6. 승인 뒤 코드/state가 바뀌었으면 재-plan한다.
7. saved plan만 apply하고 provider/API read-back과 후속 full plan을 확인한다.
8. commit/push와 원격 CI를 확인한다.

targeted apply는 예외다. 공유 stack의 unrelated pending change를 끌어오지 않는지 exact address와 destroy/change count를 확인한다.

ECR retirement는 consumer 0을 확인한 뒤 보통 두 단계로 한다.

1. `force_delete=false -> true` in-place 준비와 apply
2. 별도 승인 뒤 repo/lifecycle resource destroy

Lambda image ECR 예외와 dormant rollback consumer를 빠뜨리지 않는다.

## OnPrem host와 k3s

- 플랫폼은 `onprem` Ansible이 소유한다.
- 앱과 Application은 Argo CD가 소유한다.
- host firewall, kernel, package, k3s config는 모든 대상 노드의 role/inventory를 확인해 serial로 reconcile한다.
- shared cluster 변경 전 Argo root operation, repo-server 안정성, 다른 migration owner를 확인한다.
- disruptive ingress/API/storage 작업은 한 노드씩 수행하고 control-plane quorum과 사용자 경로를 매 단계 확인한다.
- standalone 호스트와 k3s node inventory를 혼동하지 않는다.

읽기 전용 진단:

```bash
kubectl --context dylabs-onprem get nodes -o wide
kubectl --context dylabs-onprem get pods -A
kubectl --context dylabs-onprem -n argocd get applications.argoproj.io
```

변경을 위해 로컬 `kubectl`이나 Helm을 사용하지 않는다.

## Vault와 ESO

현재 production topology:

- authoritative Vault: OnPrem 3-node Raft
- production endpoint: `https://vault.pri.sbalyd.com`
- OnPrem ESO: cluster-local active service + OnPrem AppRole
- EKS ESO: production endpoint + EKS AppRole
- EKS snapshot uploader: dedicated snapshot AppRole + existing S3/KMS/Object Lock path
- old EKS Vault: `0/0`, PVC cold source

운영 원칙:

- 두 Vault를 동시에 writable로 열지 않는다.
- old EKS PVC를 scale-up해 rollback하지 않는다.
- recovery key, KMS credential, AppRole SecretID, root token을 출력하지 않는다.
- Vault CLI 관리자 인증이 없으면 브라우저/API 우회를 만들지 않고 blocker를 보고한다.
- policy/AppRole/ESO/backup 역할을 분리한다.
- snapshot은 Job success 외에 checksum, encryption, Object Lock, freshness를 확인한다.

## DNS, Tunnel, network

Private route:

- `*.pri.sbalyd.com`
- Route53 record와 Tailscale IP/TailVIP
- EKS↔OnPrem tailnet/subnet-router 경로

Public OnPrem route:

- Cloudflare DNS
- 단일 `dylabs-onprem` tunnel
- `cloudflared` ConfigMap ingress rule
- namespace-qualified ClusterIP service

확인 항목:

- authoritative DNS와 resolver별 전파
- CNAME/A/TXT ownership
- Cloudflare proxy 상태
- tunnel connector Pod와 config revision
- origin service endpoint
- 사용자 위치의 HTTP status, redirect, TLS
- `cdn-cgi/trace`는 보조 근거이며 origin health를 대체하지 않음

Cloudflare ConfigMap은 프로세스가 자동 reload하지 않을 수 있다. 현재 chart/manifest의 checksum 또는 revision rollout 방식을 확인해 GitOps로 반영한다.

외부 API allowlist가 EKS NAT/egress IP에 묶여 있으면 OnPrem 전환 전에 신규 egress를 승인한다. DB/Redis 연결은 단순 TCP reachability뿐 아니라 실제 client handshake, timeout, MTU/SG, workload log를 확인한다.

