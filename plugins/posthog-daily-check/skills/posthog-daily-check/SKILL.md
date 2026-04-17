---
name: posthog-daily-check
description: "Use when running PostHog product health check, analytics review, anomaly detection for meloming. Triggers on: /posthog, PostHog 점검, 제품 지표 점검, 지표 이상 탐지, product health check, 데일리 제품 점검, posthog daily check, PostHog 데일리 체크"
---

# PostHog Daily Product Health Check

Meloming PostHog 프로젝트(id 205334) 기준 일일 제품 지표 점검.
주 목표: **전날 대비 이상치 탐지 + 롤아웃 변화 추적 + 계측 품질 감시**.

SRE daily check가 인프라를 보는 반면, 이건 **유저 관점 제품 건강도**를 본다.

## Procedure

### 1. DAU 추이 & 플랫폼 분포

지난 7일 DAU를 플랫폼별로 뽑아 전주 대비 이상 여부 확인.

MCP 도구: `mcp__posthog__query-run` with HogQLQuery.

```sql
SELECT toDate(timestamp) as day,
       multiIf(properties.$lib='web','web',
               properties.$lib='posthog-ios','ios',
               properties.$lib='posthog-android','android','other') as platform,
       uniq(distinct_id) as dau
FROM events
WHERE timestamp >= now() - INTERVAL 14 DAY
  AND properties.$lib IN ('web','posthog-ios','posthog-android')
GROUP BY day, platform
ORDER BY day DESC, platform
```

- **판별**: 전날 DAU가 최근 7일 평균 대비 ±20% 벗어나면 이상. 플랫폼별 별도 판별.
- **웹 DAU 기준**: 정상 1,200~2,400. iOS 15~40. Android 30~60.

### 2. Exception 추이 & 신규 에러 경로

지난 24시간 신규/증가한 예외 경로 식별.

```sql
SELECT properties.$pathname as path,
       countIf(timestamp >= now() - INTERVAL 1 DAY) as today,
       countIf(timestamp >= now() - INTERVAL 8 DAY AND timestamp < now() - INTERVAL 1 DAY) / 7 as avg_prev_7d,
       uniq(distinct_id) as users_today
FROM events
WHERE event='$exception' AND timestamp >= now() - INTERVAL 8 DAY
GROUP BY path
HAVING today > 20
ORDER BY today - avg_prev_7d DESC
LIMIT 20
```

- **신규 경로** (`avg_prev_7d = 0` AND `today >= 10`) → Critical. 최근 배포 회귀 의심.
- **2배 이상 증가** (`today / avg_prev_7d >= 2`) → Warning.
- **홈(`/`) 예외율**: `$exception` count / `$pageview` count 비교. 25% 이상이면 심각.

전체 일별 트렌드:

```sql
SELECT toDate(timestamp) as day,
       count() as exceptions,
       uniq(distinct_id) as users
FROM events
WHERE event='$exception' AND timestamp >= now() - INTERVAL 14 DAY
GROUP BY day ORDER BY day
```

### 3. Rageclick 핫스팟 변화

새로 등장한 rageclick 또는 급증한 버튼 식별.

```sql
SELECT properties.$el_text as el_text,
       properties.$pathname as path,
       countIf(timestamp >= now() - INTERVAL 1 DAY) as today,
       countIf(timestamp >= now() - INTERVAL 8 DAY AND timestamp < now() - INTERVAL 1 DAY) / 7 as avg_prev_7d,
       uniq(distinct_id) as users_today
FROM events
WHERE event='$rageclick' AND timestamp >= now() - INTERVAL 8 DAY
  AND properties.$el_text IS NOT NULL
GROUP BY el_text, path
HAVING today >= 5
ORDER BY today - avg_prev_7d DESC
LIMIT 15
```

- 유저당 평균 10회 이상인 요소는 **UX 개선 후보**.
- 신규 el_text가 급등장하면 → 최근 배포된 UI의 결함 가능성.

### 4. Feature Flag 롤아웃 변화 감지

전날 대비 flag의 true/false 비율 변화 확인.

```sql
SELECT properties.$feature_flag as flag,
       toDate(timestamp) as day,
       countIf(properties.$feature_flag_response = 'true') as t,
       countIf(properties.$feature_flag_response = 'false') as f,
       round(100.0 * countIf(properties.$feature_flag_response = 'true') / count(), 1) as pct_on
FROM events
WHERE event='$feature_flag_called' AND timestamp >= now() - INTERVAL 3 DAY
GROUP BY flag, day
ORDER BY flag, day DESC
```

- 전날 대비 `pct_on` ±20%p 변화 → 롤아웃 변경 발생. 의도적인지 확인.
- `null` 응답만 나오는 flag → 설정 오류 (`home-headerless-renewal` 같은 케이스).
- 30일 연속 0% ON → stale flag 후보 (제거 검토).

Flag 메타데이터:
```
mcp__posthog__feature-flag-get-all with active=true
```

### 5. 계측 품질 점검 (Dead tracking)

30일 연속 미발화 커스텀 이벤트는 계측 코드가 살아있는데 UI가 없거나 반대.

```sql
SELECT event, max(timestamp) as last_seen,
       count() as total_30d
FROM events
WHERE timestamp >= now() - INTERVAL 30 DAY
  AND NOT startsWith(event, '$')
  AND event NOT IN ('Application Opened','Application Backgrounded','Application Installed',
                    'Application Updated','Deep Link Opened','survey shown','survey dismissed','survey sent')
GROUP BY event
ORDER BY total_30d ASC
LIMIT 20
```

- `total_30d = 0` 이면 **완전 dead** — 코드 제거 후보.
- `last_seen < now - 7 days` 이면 최근 미발화 — 관련 화면 상태 확인.

그리고 iOS의 $screen 값 점검 — 원시 SwiftUI 클래스명이면 계측 gap:

```sql
SELECT properties.$screen_name as screen, count() as views, uniq(distinct_id) as users
FROM events
WHERE event='$screen' AND properties.$lib='posthog-ios'
  AND timestamp >= now() - INTERVAL 1 DAY
GROUP BY screen ORDER BY views DESC LIMIT 10
```

`UIHostingController<...>`, `PresentationHostingController<...>` 등이 상위에 있으면 **수동 screen() 호출 누락**.

### 6. 로그인/가입 funnel 건강도

```sql
-- 지난 7일 auth/login 방문자 수
SELECT toDate(timestamp) as day,
       uniq(distinct_id) as login_page_visitors
FROM events
WHERE event='$pageview' AND properties.$pathname = '/auth/login'
  AND timestamp >= now() - INTERVAL 7 DAY
GROUP BY day ORDER BY day
```

- 방문자 급감 → 유입 문제 or 로그인 버튼 링크 고장 가능성.
- 커스텀 `login_succeeded` 이벤트가 생기면 같이 쿼리.

### 7. 유입 채널 변화

```sql
SELECT properties.$referring_domain as referrer,
       countIf(timestamp >= now() - INTERVAL 1 DAY) as today_sessions,
       countIf(timestamp >= now() - INTERVAL 8 DAY AND timestamp < now() - INTERVAL 1 DAY) / 7 as avg_prev_7d
FROM events
WHERE event='$pageview' AND timestamp >= now() - INTERVAL 8 DAY
  AND properties.$referring_domain IS NOT NULL
GROUP BY referrer
HAVING today_sessions >= 50
ORDER BY today_sessions - avg_prev_7d DESC
LIMIT 10
```

- 치지직/SOOP/Twitter 같은 주요 채널 급락 → 유입 경로 점검.
- 새로 등장한 referrer가 볼륨 크면 바이럴 유입 or scraper.

### 8. 이상 식별 기준

| 항목 | 경고 | 심각 |
|------|------|------|
| DAU 변화 | ±20% | ±40% |
| Exception 일건수 | 7일 평균 대비 +50% | 전일 대비 +70% |
| 신규 예외 경로 | 1개 이상 | 일 10건 이상 |
| Rageclick 신규 핫스팟 | 유저당 3회 이상 | 10회 이상 |
| Flag rollout 변화 | ±20%p | ±50%p |
| Dead event | 30일 발화 0 | 60일 발화 0 |
| iOS screen 원시 클래스명 | 상위 1개 | 상위 3개 |

### 9. 결과 리포트

```
## PostHog Daily Report — {YYYY-MM-DD}

### 유저 활성도
- Web DAU: {N} ({전날 대비 %})
- iOS DAU: {N} / Android DAU: {N}
- 이상 신호: {목록 또는 "없음"}

### Exception
- 일 건수: {N} (7일 평균 대비 %)
- 신규 경로: {path 목록 또는 "없음"}
- 급증 경로: {path 목록 또는 "없음"}
- 홈 예외율: {pct}%

### Rageclick
- 신규/급증 핫스팟: {element text @ path 목록}
- 개선 제안: {구체 UI 변경}

### Feature Flag
- 롤아웃 변화 감지: {flag: X% → Y%, 목록 또는 "없음"}
- 설정 오류 의심: {목록}
- Stale 후보: {30일 0% ON flag 목록}

### 계측 품질
- Dead event: {목록 또는 "없음"}
- iOS screen 계측 gap: {원시 클래스명 상위}

### 유입
- 전일 대비 급변 referrer: {목록}

### 즉시 대응 필요
- {항목 또는 "없음"}

### 권장 조치
- {Action Items with repo mapping}
```

### 10. last_run 업데이트

점검 완료 후 반드시 `DAILY-TASKS.md`의 `PostHog Daily Check` 항목 last_run 갱신:

```
파일: ~/.claude/projects/-Users-hyeonwoo-DEV/memory/DAILY-TASKS.md
변경: `- **last_run**: {이전 날짜}` → `- **last_run**: {오늘 YYYY-MM-DD}`
```

## Notes

- SRE daily check와 병렬 실행 가능 (다른 데이터 소스).
- PostHog MCP 도구는 필요 시 ToolSearch로 `select:mcp__posthog__query-run,mcp__posthog__feature-flag-get-all,mcp__posthog__event-definitions-list` 로 로드.
- 쿼리는 HogQL (ClickHouse 기반). `toTimeZone` (대소문자 주의), `properties.$prop_name` 문법.
- 예외 message/type이 모두 null이면 posthog-js 설정 누락 — P0 이슈로 보고.
- Default project가 Meloming (205334). 다른 project 조사 시 `switch-project` 선행.
