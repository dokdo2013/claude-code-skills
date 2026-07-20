# Monitoring, incidents, and completion

## 현재 관측 구조

2026-07-21 live 확인 기준:

- EKS: Prometheus, Grafana, Loki, Alertmanager, Fluent Bit와 exporters가 실행 중이다.
- OnPrem: 자체 Prometheus, Grafana, Loki, Alertmanager가 실행 중이다.
- live Thanos Pod는 양 클러스터에 없다. 리포에는 과거·계획·stale datasource 문자열이 남을 수 있으므로 문자열만 보고 현재 topology를 단정하지 않는다.
- alert/rule/dashboard/exporter는 주로 `monitoring-gitops`, stack 설치/values는 `k8s-manifests`가 소유한다.
- Alertmanager routing은 양 클러스터의 실제 config와 rule owner를 확인한다. “EKS만 authoritative” 같은 과거 규칙을 재사용하지 않는다.

이 구조는 변경 가능하다. 작업 전에 datasource API, scrape target, rule evaluator, Alertmanager route를 live로 확인한다.

## 모니터링 변경

PrometheusRule:

- 기존 kustomization과 label contract를 확인한다.
- recording rule, dashboard, KEDA/HPA가 같은 label을 기대하는지 검색한다.
- `warning`/`critical`은 실제 사용자 영향과 조치 가능성에 맞춘다.
- alert noise를 줄이기 위해 오류 수만 보지 말고 지속시간, rate, drop, freshness, saturation을 사용한다.

Grafana:

- dashboard provisioning 방식이 sidecar인지 static mount인지 live stack에서 확인한다.
- datasource UID를 dashboard JSON과 live datasource에서 대조한다.
- Pod Ready만으로 datasource query와 OAuth/권한을 정상 판정하지 않는다.

Alertmanager:

- Slack/incident.io route는 현재 config와 secret Ready를 확인한다.
- 테스트 알림은 외부 side effect이므로 사용자 요청 범위와 maintenance window를 확인한다.

## 읽기 전용 진단

```bash
kubectl --context <cluster> -n <namespace> get pods -o wide
kubectl --context <cluster> -n <namespace> describe pod <pod>
kubectl --context <cluster> -n <namespace> logs <pod> --tail=200
kubectl --context <cluster> -n <namespace> logs <pod> --previous --tail=200
kubectl --context <cluster> -n <namespace> get events --sort-by='.lastTimestamp'
kubectl --context <cluster> top pods -n <namespace>
kubectl --context <cluster> top nodes
kubectl --context dylabs-onprem -n argocd get application <app> -o yaml
```

진단 요청에서는 자동 수정하지 않는다. P0라도 영향과 안전한 조치를 먼저 보고하고 승인 범위를 확인한다.

## 증상별 확인 순서

| 증상 | 먼저 확인 | 흔한 원인 |
|---|---|---|
| CrashLoopBackOff | previous log, env source, Secret, dependency | app error, secret drift, DB/Redis/Vault timeout |
| OOMKilled | Last State, limit, 장기 peak, replica/HPA | limit 부족, leak, burst, 잘못된 request |
| Pending | events, selector, affinity, taint, PVC | architecture/placement mismatch, capacity, volume |
| ImagePullBackOff | exact image, pull Secret, registry auth, remote tag | nonexistent tag, missing GHCR Secret, architecture |
| Argo Unknown/ComparisonError | repo-server, render error, webhook/API | values schema, missing path, admission/network |
| OutOfSync | diff와 manager/owner | defaulted field, 중복 owner, 실제 source drift |
| ExternalSecret 오류 | status, event, store, policy, AppRole | path/policy/TTL/endpoint |
| Tunnel 404/503 | DNS, ordered ingress rule, service endpoints | hostname 누락, namespace/service 오류, stale config |
| Redis/DB hang | client handshake, connection pool, SG/MTU, log | cross-cluster path, max connections, ready-check |
| HPA replica 증가 | CPU request/target, 실제 load, stabilization | 정상 burst, request 과소, scale-down delay |

Argo `OutOfSync`이 Healthy라고 해서 임의 sync/patch하지 않는다. 다른 세션의 진행 중 변경, defaulted field, shared owner일 수 있다.

## 용량과 비용

- node 수보다 workload request, actual peak, PDB, affinity, topology spread, HPA/KEDA를 먼저 본다.
- 장기 peak와 p95/p99, restart/OOM/throttling, rollout headroom을 함께 본다.
- CPU limit은 무조건 요구하지 않는다. CPU request, memory request/limit을 우선 검토한다.
- Karpenter consolidation을 위해 PDB/affinity를 약화할 때 가용성 trade-off를 명시한다.
- managed nodegroup/bootstrap 역할을 제거하기 전 Karpenter/CoreDNS/CSI의 복구 경로를 확인한다.
- 비용 계산은 EC2, EBS, LB, NAT, transfer, control plane 등 포함 범위를 명시한다.

## 배포/운영 완료 체크

- CI 성공
- GitOps revision/tag/digest
- Argo sync/health/operation
- old Pod 0, expected new Pod Ready
- actual Pod image/digest와 source SHA
- restart/OOM/Pending/event
- dependency log와 health
- actual DNS/Tunnel/Ingress/API/browser path
- remote branch와 commit

한 항목이라도 미확인이면 “CI 완료”, “GitOps 반영 중”, “롤아웃 확인 대기”처럼 정확한 단계만 말한다.

공유 인프라 작업은 최종적으로 다음도 확인한다.

- root/platform App의 Unknown/ComparisonError/Running operation
- 다른 destination App의 회귀
- shared Namespace/Secret/route
- Terraform state와 provider read-back
- rollback asset와 정리 시점

