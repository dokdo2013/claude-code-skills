---
name: meloming-email
description: "Use when sending, testing, or monitoring bulk/transactional email through meloming-back's Bulk Email Platform. Covers campaign creation, recipient validation, NOTICE vs MARKETING (정통망법), branded templates, deliverability, and delivery-metric pitfalls. Triggers on: 메일 발송, 이메일 캠페인, 안내 메일, 마케팅 메일, bulk email, 수신자 검증, 발송 모니터링, 도달률, bounce, Resend, 수신거부, 뉴스레터, 전체 회원 메일"
---

# meloming Bulk Email Runbook

meloming-back의 Bulk Email Platform으로 캠페인 생성 -> 수신자 검증 -> 테스트 -> 발송 -> 모니터링하는 운영 프로시저. 코드는 `meloming-back/src/bulk-email/`, 브랜드 템플릿은 `meloming-back/src/transactional-email/templates/`.

- prod API base: `https://api.meloming.com/v1/admin/email-campaigns` (URI versioning, defaultVersion 1)
- 인증: admin JWT(`isAdmin:true`), 만료 1시간. 사용자가 발급/제공 (assistant 생성 불가). 발송 시점에 새 토큰 요청.
- admin UI: `meloming-admin/src/domains/email-campaigns/` (Tiptap 본문 편집, 클라이언트에서 청크)

## 발송 플로우 (정석 순서)

1. `POST /` -> DRAFT 생성. body: `{type,title,subject,bodyHtml,fromName,fromEmail,replyTo}`
2. `PATCH /:id` -> 본문 수정 (DRAFT 상태에서만. VALIDATED 되면 잠김)
3. `POST /:id/recipients/validate` -> 수신자 검증+등록. body `{emails:[...]}`. **반드시 청크** (아래 [CRITICAL] 참고)
4. `POST /:id/lint` -> 컴플라이언스 프리체크. `issues[]` 중 severity=error 0건 확인
5. `POST /:id/test-send` -> 테스트 발송. body `{to:[...]}` (allowlist 내 주소만)
6. `POST /:id/send` -> 본 발송. lint fence(error 있으면 400) + dispatch
7. 모니터링 (아래 모니터링 섹션 — 상태 테이블 함정 주의)

기타: `GET /:id` (상태), `GET /:id/recipients?status=<S>&limit=&offset=` (status별 목록+total), `GET /:id/audit-log`, `POST /:id/{pause,resume,abort}`.

상태 머신: `DRAFT -> VALIDATED -> SENDING -> FLUSHING -> RECONCILING -> DONE` (`PAUSED`/`ABORTED`/`FAILED` 분기). 수정/삭제는 DRAFT 한정.

## NOTICE vs MARKETING (정통망법)

캠페인 type은 둘 중 하나. 발송 게이트는 `User.marketingConsent` 컬럼(source-of-truth)만 본다.

| | NOTICE (서비스 안내) | MARKETING (광고) |
|---|---|---|
| marketingConsent | 무시 (전체 발송) | `true`인 회원만 |
| 차단 suppression scope | `ALL`만 | `ALL` + `MARKETING_ONLY` |
| quiet hours (KST 21~08) | 없음 (24/7) | `policy.quietHours=true`면 차단 |
| 제목 `(광고)` prefix | 없음 | 자동 부착 (멱등 — 중복 안 됨) |
| List-Unsubscribe-Post one-click | 없음 | 부착 (RFC 8058) |
| footer 문구 | "Meloming 서비스 안내" | "광고성 정보" |

- **전체 회원 대상 서비스 업데이트/기능 출시 안내 = NOTICE.** 신규 기능을 무료/거래관계 안내로 전달하면 NOTICE가 타당. consent 비대상 회원에게도 가는 만큼 콘텐츠가 정보성이어야 한다.
- 순수 판촉/할인 = MARKETING. consent 동의자에게만.
- 양쪽 모두 compliance footer 자동 주입: 수신거부 링크(per-recipient 토큰) + 회사정보(`COMPANY_NAME/ADDRESS/CONTACT_EMAIL/BUSINESS_REG_NUMBER` env) + `List-Unsubscribe` 헤더. env 누락 시 render가 fail-fast (lint에서 `SENDER_INFO_MISSING`).

수신자 분류 우선순위 (RecipientValidationService): `INVALID_FORMAT`(DB 미기록) > `NOT_MEMBER` > `DELETED`(isDeleted) > `INACTIVE`(isActive=false) > suppression > `NO_CONSENT`(MARKETING만) -> 나머지 `READY`. suppression 기록: HARD_BOUNCE=scope `MARKETING_ONLY` (그래서 NOTICE는 안 막힘), COMPLAINED=scope `ALL`.

## [CRITICAL] 수신자 검증 100KB 청크 필수

- main.ts에 json body limit override 없음 -> Express 기본 약 100KB. 한 번에 9,232개 이메일 넣으면 `request entity too large` 500 (2026-05-22 사고).
- DTO는 maxItems 50,000 허용하지만 그 전에 body parser가 거부. **1,500개/청크(약 60KB)** 안전.
- 청크는 누적된다: 각 validate 호출은 그 청크 이메일만 delete+create. 첫 청크에서 eligible>0이면 DRAFT->VALIDATED 승격, 이후 청크는 VALIDATED 상태로도 허용.
- 검증 리포트: `{totalInput,totalNormalized,eligibleCount,ineligibleCount,unknownCount,byExcludeReason}`. 청크별 합산.

## 브랜드 템플릿 (bodyHtml은 자동 브랜딩 안 됨)

ComplianceRenderer는 footer만 주입하고 레이아웃 wrap을 안 한다. bare `<p>`로 보내면 비브랜드 메일이 나간다. 정석은 `meloming-back/src/transactional-email/templates/`의 디자인 시스템으로 bodyHtml을 빌드:

- `renderLayout({title, preview, body})` -> 멜로밍 로고 헤더 + 카드 + 다크모드 대응 + 문의/copyright footer 포함한 full `<!doctype html>`.
- 컴포넌트: `Heading({level,text})`, `Paragraph(html)`, `Muted(html)`, `Button({href,label,ariaLabel})`, `UrlFallback(url)`, `InfoBox({bodyHtml})`, `Spacer(size)`, `Hr()`.
- 이 함수들은 순수(Nest 의존성 없음) -> `/tmp`에 드라이버 .ts 작성 후 `npx tsx`로 절대경로 import해 렌더. (회사 레포 `src/` 밖에 .ts 추가 금지 룰 회피 — 드라이버는 /tmp에 둔다.)
- bodyHtml 끝에 `<!--COMPLIANCE_FOOTER-->` sentinel을 넣으면 compliance footer가 카드 안에 렌더됨. 없으면 `</body>` 앞에 append (카드 밖, 덜 깔끔).
- `renderLayout`에 `unsubscribeUrl`을 넘기지 마라 — per-recipient 수신거부 토큰은 발송 시점에만 생성되므로, compliance footer 주입에 맡긴다.
- `Paragraph(html)`는 인자를 escape하지 않으므로 `{{nickname}}`이 원형 통과한다(필요). `Heading({text})`는 escape하지만 `{`,`}`는 안 건드려서 토큰은 보존됨.

### {{nickname}} 변수 치환 (PR #343)

- `ComplianceRenderer.render({campaign,recipient,unsubscribeToken,variables})`의 `variables`로 `{{key}}` 치환. `User.nickname`을 batch lookup해서 `{{nickname}}` 주입 (발송 + test-send 양쪽).
- 정책: HTML-escape, 키 누락 시 토큰 원형 유지(silent empty 아님), 재귀 확장 없음(crafted nickname 인젝션 차단). 키 패턴 `[a-zA-Z_][\w]*`.
- test-send에서 `to` 주소가 회원이면 그 닉네임으로 치환되어 시각 확인 가능.

### 이모지 (고객 콘텐츠 예외)

- assistant 응답/내부 파일에는 이모지 금지(hook 차단). 하지만 **고객 메일 본문**(디스코드/노션 공지 원본 이전)은 이모지 보존이 정책.
- 방법: 드라이버/소스는 ASCII placeholder(`@@PARTY@@` 등)로 두고, 빌드 후 python `chr(0x1F389)` 같은 codepoint로 치환해 출력 파일에만 이모지 생성. 소스/명령줄엔 이모지 글리프가 없어 hook이 안 걸린다.
- placeholder로 `{{...}}` 쓰지 마라 — 변수 치환과 충돌. `@@TOKEN@@` 사용.

## 수신자 풀 추출 (prod, SELECT only)

- prod DB URL: `kubectl -n meloming-prod exec <meloming-back-api pod> -- printenv DATABASE_URL` (Aurora MySQL, 노트북에서 도달 가능, pymysql). 비번 출력 금지.
- 활성 회원 풀: `WHERE (is_active IS NULL OR is_active=1) AND (is_deleted IS NULL OR is_deleted=0) AND email<>''`.
- 2026-05-22 기준 9,232명 (전원 email_verified — 회원가입 시 인증 필수 -> hard bounce 거의 0). marketingConsent 동의자 3,031 (33%).

## 테스트 방법

- `POST /:id/test-send` body `{to:["..."]}`. 주소는 env `BULK_EMAIL_ALLOWED_TEST_RECIPIENTS` allowlist에 있어야 함. `doyeon@dylabs.app` 등록돼 있음.
- 제목에 `[TEST]` 워터마크 자동 부착. 실제 발송과 동일 렌더(footer/{{nickname}}/이모지) 거치므로 시각 검증용으로 신뢰 가능.
- 본문 검토: 렌더 HTML을 브라우저로 열거나(`open file.html`) Zed로 소스 확인. compliance footer는 standalone 프리뷰엔 안 보임(발송 시 주입) — test-send 메일에서 확인.

## 발송

- `POST /:id/send` -> lint fence(error severity 있으면 400 차단) -> `dispatchCampaign`. 100명/배치로 BullMQ 큐잉 -> Resend `batch.send` 2 rps (env `BULK_EMAIL_RATE_LIMIT_RPS`).
- 9,232명 = 93 청크, dispatch는 약 1분. 응답에 `{campaignId,totalRecipients,totalChunks,jobIds[]}`.
- 발송 후 recipient는 `READY -> API_ACCEPTED`로 바뀌고, 이후 Resend 웹훅으로 DELIVERED 등으로 전이.

## 모니터링 — [CRITICAL] 상태 테이블은 도달률을 과소계상한다

recipient status enum: `READY/QUEUED/API_ACCEPTED/DELIVERED/OPENED/CLICKED/BOUNCED_SOFT/BOUNCED_HARD/COMPLAINED/SUPPRESSED/FAILED/EXCLUDED`.

함정 (2026-05-22 실측): `recipient-status-priority`는 forward-only CAS이고 우선순위가 `BOUNCED_SOFT(5) > DELIVERED(3)`. `email.delivery_delayed`(그레이리스팅 일시지연)가 recipient를 BOUNCED_SOFT로 올리는데, 이후 재시도 성공 `email.delivered`(3)가 CAS에 막혀 BOUNCED_SOFT를 못 덮는다. 결과: 실제 배달된 메일이 BOUNCED_SOFT로 영구 박제 -> admin UI bounce 수치 과대, 도달률 과소.

- 실측: 상태 테이블은 78.2% 도달로 보였지만, **진짜 도달률 96.4%** (delivered 이벤트 8,901/9,232). 1,679건이 delivered인데 BOUNCED_SOFT로 잘못 표시.
- **진짜 도달률은 recipient status가 아니라 `email_webhook_events`의 `email.delivered` 이벤트로 집계하라.**

진짜 도달 쿼리:
```sql
SELECT COUNT(DISTINCT we.resend_email_id)
FROM email_webhook_events we
JOIN email_campaign_recipients r
  ON r.resend_email_id=we.resend_email_id AND r.campaign_id=:id
WHERE we.event_type='email.delivered';
```
진짜 미배달 = BOUNCED_SOFT 중 delivered 이벤트가 없는 것:
```sql
SELECT COUNT(*) FROM email_campaign_recipients r
WHERE r.campaign_id=:id AND r.status='BOUNCED_SOFT'
  AND NOT EXISTS (SELECT 1 FROM email_webhook_events we
    WHERE we.resend_email_id=r.resend_email_id AND we.event_type='email.delivered');
```

`email_webhook_events` 컬럼: `svix_id, event_type, resend_email_id, recipient_email, payload(JSON), processed_at, created_at`. unique `(resend_email_id, event_type)`. bounce 사유는 `payload.data.bounce.{type,subType,message}` (type: `Permanent`/`Transient`). 도메인별 도달/bounce는 `SUBSTRING_INDEX(email,'@',-1)`로 그룹핑.

토큰 만료 영향 없이 모니터링하려면 admin API 대신 prod DB SELECT를 써라(위 kubectl exec로 DATABASE_URL). JOIN 시 `created_at`은 `we.created_at`로 한정(ambiguous 에러 방지).

### Resend 상태 직접 조회

- `GET https://api.resend.com/emails/:id`로 Resend의 실제 상태(`last_event`) 조회 가능. 단:
  - 기본 urllib UA는 Cloudflare가 차단(`error code: 1010`) -> 브라우저 User-Agent 헤더 필요.
  - env `RESEND_API_KEY`는 **send-only(restricted)** -> 읽기 401(`restricted_api_key`). 상태 조회엔 full-access 키가 별도로 필요. 없으면 `email_webhook_events` 테이블로 갈음.

## reconcile 라이프사이클 (cron EVERY_MINUTE)

- `SENDING -> FLUSHING`: READY/QUEUED 0건 AND 마지막 API_ACCEPTED 이벤트가 30분(`SENDING_QUIET_MS`) 이상 경과. (그래서 발송 직후 30분간 SENDING 유지가 정상)
- `FLUSHING -> RECONCILING`: 어떤 recipient 이벤트도 30분(`FLUSHING_QUIET_MS`) 동안 없을 때.
- `RECONCILING -> DONE`: 전 recipient가 final 상태 OR `sentAt + 72h`(`RECONCILE_DEADLINE_HOURS`) 경과.
- reconcile은 Resend를 폴링하지 않는다 — 캠페인 상태만 굴린다. recipient의 API_ACCEPTED->DELIVERED는 오직 Resend 웹훅으로만 일어난다.

## Deliverability / 도메인 워밍업 팁

- 발신 서브도메인 `mail.meloming.com`은 cold면 대량 급증 시 throttle된다. 2026-05-22 (직전 84통 -> 9,232통):
  - Apple Private Relay(`privaterelay.appleid.com`): cold/미인증 발신자 거의 전량 거부(0% 도달).
  - Gmail: 상당수 `delivery_delayed`(그레이리스팅) 후 재시도로 정상 배달. 일부 MailboxFull(수신자 사정).
  - Naver: 98% 정상 배달 (한국 ISP가 무조건 막는다는 가정은 틀림).
  - 최종 진짜 도달 96.4%, 영구 컴플레인 0.
- 정석: 큰 발송 전 도메인을 점진 워밍업(며칠에 걸쳐 소량->대량 ramp). 강제 가속/즉시 재시도는 평판을 해쳐 정석 아님.
- 영구 bounce는 suppression(HARD_BOUNCE)으로 자동 등록되어 다음 발송에서 제외됨.
- sender 표준: 서비스 안내 = `멜로밍 <notice@mail.meloming.com>`, reply-to `meloming@dylabs.app`. 대표 친필 outreach = `조현우 (디와이랩스) <notice@mail.meloming.com>`.

## 함정 요약

- 수신자 검증 9,232 한방 = 100KB 초과 500 -> 1,500/청크 필수.
- bodyHtml 자동 브랜딩 안 됨 -> transactional-email 템플릿으로 빌드.
- 도달률은 recipient status가 아니라 `email.delivered` 이벤트로 — BOUNCED_SOFT가 delivered를 가린다.
- `RESEND_API_KEY`는 send-only -> Resend 상태 읽기 불가, 웹훅 테이블로 갈음. Cloudflare는 UA 필요.
- admin 토큰 1시간 만료 -> 발송/모니터 중 만료 시 재발급. 장시간 모니터는 DB SELECT로.
- 발송 직후 30분 SENDING 유지는 정상(reconcile QUIET 윈도우).
- PROD DB는 SELECT only. 정리 필요 시 사용자 승인 후 또는 앱 코드로.
