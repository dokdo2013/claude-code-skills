---
name: sre-daily-check
description: "Use when running SRE daily health check, system inspection, or cluster status review. Triggers on: /sre, SRE 점검, 시스템 점검, 클러스터 상태, daily health check, 데일리 헬스체크, 인프라 점검, pod 상태, 노드 상태"
---

# SRE Daily Health Check

EKS 클러스터 `dylabs-prod-eks-main` 대상 일일 시스템 점검.

## Procedure

### 1. 클러스터 노드/Pod 상태 (병렬 실행)

```bash
# 노드 리소스 사용률
kubectl top nodes

# 노드 상태 (NotReady, SchedulingDisabled 등 이상 확인)
kubectl get nodes -o wide

# 비정상 Pod 확인 (Running/Succeeded 아닌 것)
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# 최근 30분 Warning 이벤트
kubectl get events -A --sort-by='.lastTimestamp' --field-selector=type=Warning | tail -30
```

위 4개 명령은 **병렬 실행** 가능.

### 2. Alertmanager 알람 히스토리

```bash
# port-forward (백그라운드)
kubectl port-forward -n monitoring svc/alertmanager-operated 9093:9093 &
sleep 2

# 현재 firing 알람
curl -s 'http://localhost:9093/api/v2/alerts?silenced=false&inhibited=false' | jq '[.[] | select(.status.state=="active")] | length'
curl -s 'http://localhost:9093/api/v2/alerts?silenced=false&inhibited=false' | jq '.[] | select(.status.state=="active") | {alertname: .labels.alertname, severity: .labels.severity, startsAt: .startsAt, summary: .annotations.summary}'

# Silence 목록
curl -s 'http://localhost:9093/api/v2/silences' | jq '.[] | select(.status.state=="active") | {id: .id, matchers: .matchers, createdBy: .createdBy, endsAt: .endsAt}'

kill %1 2>/dev/null
```

**False Alert 판별**: 5분 이내 firing→resolved 반복, 동일 알람 24시간 내 3회 이상.

### 3. Prometheus 핵심 지표

```bash
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090 &
sleep 2
```

쿼리들 (병렬 실행 가능):

```bash
# 5xx 에러율 (1h)
curl -s 'http://localhost:9090/api/v1/query?query=sum(rate(http_requests_total{code=~"5.."}[1h])) by (service)' | jq '.data.result[] | {service: .metric.service, rate: .value[1]}'

# 응답 p95 (1h)
curl -s 'http://localhost:9090/api/v1/query?query=histogram_quantile(0.95,sum(rate(http_request_duration_seconds_bucket[1h])) by (le,service))' | jq '.data.result[] | {service: .metric.service, p95_seconds: .value[1]}'
```

Prometheus 쿼리가 빈 결과면 `kubectl top pods` fallback:

```bash
kubectl top pods -n meloming-prod --sort-by=memory | head -15
kubectl top pods -n meloming-prod --sort-by=cpu | head -15
```

PVC 사용률:

```bash
curl -s 'http://localhost:9090/api/v1/query?query=(kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100 > 70' | jq '.data.result[] | {pvc: .metric.persistentvolumeclaim, namespace: .metric.namespace, used_pct: .value[1]}'
```

```bash
kill %1 2>/dev/null
```

### 4. 위험 수치 식별

- 메모리/CPU 80%+ → 스케일 업/아웃 검토
- 5xx 급증 → `kubectl logs` 확인
- OOMKilled → `kubectl describe pod` + limits 조정
- Pending Pod → 스케줄링 실패 원인 (리소스 부족, nodeSelector 등)
- CrashLoopBackOff → 최근 로그 확인, 배포 상태 점검

### 5. 리포트 출력

```
## SRE Daily Report — {YYYY-MM-DD}

### 클러스터 상태
- 노드: {N}개 정상 / {M}개 이상
- 비정상 Pod: {목록 또는 "없음"}

### 알람 분석
- 현재 Firing: {N}건
- False Alert 후보: {목록}
- 정리 제안: {monitoring-gitops 수정 사항}

### 주요 지표
- 5xx 에러: {서비스별 현황}
- 지연 이상: {p95 > 1s 서비스}
- 리소스 경고: {메모리/CPU 80%+ Pod}

### 즉시 대응 필요
- {항목 또는 "없음"}

### 권장 조치
- {항목 또는 "특이사항 없음"}
```

### 6. last_run 업데이트

점검 완료 후 반드시 DAILY-TASKS.md의 `SRE Daily Health Check` 항목 `last_run`을 오늘 날짜로 업데이트:

```
파일: ~/.claude/projects/-Users-hyeonwoo-DEV/memory/DAILY-TASKS.md
변경: `- **last_run**: {이전 날짜}` → `- **last_run**: {오늘 YYYY-MM-DD}`
```
