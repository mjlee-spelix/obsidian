---
tags: [프로젝트, dify, AI-Agent, audit, dept-activity]
type: inbox/note
date: 2026-05-14
related:
  - "[[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md]]"
  - "[[3. 프로젝트/spx-agent/references/audit-details-spec.md]]"
---

# 부서별 API 호출 집계 — `api_call` 안 쓰고 `message_send`+`workflow_execute` 두 개 쓰는 맥락

## TL;DR

부서별 활동 표의 "호출 수/토큰" 컬럼은 **`action='api_call'` (nginx 출처) 안 쓰고, `message_send`+`workflow_execute` (DB 폴링 출처) 두 개를 합산**해서 만든다. 같은 API 호출이 두 채널에 동시 기록되는데, DB 채널에만 `app_id`가 들어 있어 부서 분류가 가능하기 때문이다.

## 배경 — 같은 호출이 audit_events에 두 번 들어온다

외부 클라이언트가 `/v1/chat-messages` 같은 API를 호출하면 Dify 내부 흐름:

```
외부 클라이언트
   ↓
 nginx (access.log 기록) ─────► log-watcher.ts ─► action='api_call'  ❌ app_id 없음
   ↓
 Dify API (Flask)
   ↓
 PostgreSQL `messages` INSERT ─► messages.ts ───► action='message_send' ✅ app_id 있음
```

즉 audit_events에 **동일 호출이 2건** 들어옴:
- `action='api_call'` — HTTP 레벨, nginx 출처
- `action='message_send'` (또는 `workflow_execute`) — DB 레벨, Dify DB 출처

## 왜 `api_call`로는 부서 분류 불가

`api_call`은 nginx access.log에서 수집되는데, 표준 combined 로그 포맷에는 **앱 식별 정보가 없다**:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';
```

- Dify API는 `Authorization: Bearer app-xxxxx` 헤더로 앱을 특정
- 하지만 `$http_authorization`은 log_format에 **없음**
- URI(`/v1/chat-messages`)도 앱 ID를 포함하지 않음
- 결과: `details.targetAppId` 없음 → `resource_ownership` JOIN 불가 → 부서 분류 불가

해결하려면 log_format에 `$http_authorization` 추가가 필요한데, **앱 API 키가 평문으로 로그 파일에 쌓이는 보안 리스크**라 회사 정책상 통과 어려움.

## 왜 `message_send`/`workflow_execute`는 부서 분류 가능

DB 폴링 collector가 `messages` / `workflow_runs` 테이블을 직접 읽으면서 `app_id`를 같이 가져온다 (`collectors/messages.ts:39`):

```sql
SELECT
  m.app_id::text AS app_id,        -- ✅ 앱 ID 있음
  m.from_source,                    -- 'api' / 'console' / 'web-app' 구분
  m.invoke_from,                    -- 'service-api' 등
  ...
FROM messages m
LEFT JOIN apps app ON m.app_id = app.id
```

저장 시 `targetId: r.app_id`로 audit_events top-level 컬럼에 들어가므로, `resource_ownership.owner_department_id`와 JOIN해서 부서 분류 자연스럽게 됨.

## 채널별 가용성 정리

| API 종류 | DB 행 생성? | DB collector 잡음? | app_id 가용? | 부서 분류 |
|---------|------------|------------------|-------------|---------|
| `/v1/chat-messages` | ✅ `messages` | ✅ messages.ts | ✅ | ✅ |
| `/v1/completion-messages` | ✅ `messages` | ✅ messages.ts | ✅ | ✅ |
| `/v1/workflows/run` | ✅ `workflow_runs` | ✅ workflow-runs.ts | ✅ | ⚠️ (triggered_from 미수집 — 콘솔/디버깅과 구분 어려움) |
| 인증 실패 (401/403) | ❌ | ❌ | nginx만 잡음 | ❌ |
| `/v1/datasets/*` (KB API) | 작업별 | 일부만 | ⚠️ | 부분적 |

## `api_call`은 그럼 어디 쓰나

부서별 활동 표 미포함. 대신 **운영팀 시스템 모니터링 용도**:

| 용도 | 무엇을 보나 |
|------|-----------|
| 보안 메트릭 | 401/403 패턴, rate_limit, 의심 IP |
| HTTP 메트릭 | 5xx 비율, 응답시간 분포, 전체 트래픽 양 |

요약: `api_call`은 "누가 무엇을 했나"가 아니라 "트래픽이 어떻게 들어왔나"를 보는 용도. 부서/앱 차원이 필요한 본 표와는 결이 다름.

## 그래서 보강해야 할 collector

부서별 활동 표를 audit 마트로 본가동하려면 P0 보강 필요:

| collector | 추가할 키 | SQL 변경 | 방어 Harness |
|-----------|---------|---------|------------|
| **messages.ts** | `appMode` | `SELECT app.mode AS app_mode` | H-DASH-01 (ADVANCED_CHAT 이중카운트) |
| **workflow-runs.ts** | `appMode` | `SELECT app.mode AS app_mode` | H-DASH-01 |
| **workflow-runs.ts** | `triggeredFrom` | `SELECT wr.triggered_from` | H-DASH-03 (디버깅 필터) |

> messages.ts는 이미 `invokeFrom` 수집 중이라 디버깅 필터는 추가 작업 없음. `appMode`만 P0.
> workflow-runs.ts는 `appMode` + `triggeredFrom` 둘 다 추가 필요.

## requirements.md 89-91번 줄 재해석

```
**api_call (nginx, REST API 외부 호출):**
- `details.targetAppId` 없음 → audit_events 단계에서는 부서 분류 불가
- ⚠️ PM 확인 항목: 부서별 활동 표에 nginx API 호출을 포함할지 여부 결정 필요
```

이 줄의 `api_call`은 **`action='api_call'` (nginx 출처) 이벤트 자체**를 가리키는 것이지, "외부에서 API로 호출한 행위 전체"가 아니다. chat/workflow 앱의 API 호출은 별도 채널(`message_send`/`workflow_execute`)로 이미 잡히고 있고, 거기서 부서 분류가 된다.

문구를 명확히 하려면:

> chat/completion/workflow 앱의 API 호출은 message_send/workflow_execute로 정상 집계됨.
> `action='api_call'` (nginx HTTP 레벨) 이벤트 자체는 본 표 미포함 — 보안/HTTP 메트릭 별도 화면에서 활용.

## TODO

- [ ] requirements.md 89-91번 줄 문구 명확화 (PM 확인 항목 → 결정사항으로 격상 가능)
- [ ] collector 보강 PR — messages.ts `appMode`, workflow-runs.ts `appMode`+`triggeredFrom`
- [ ] dept-activity 표 캡션에 "콘솔/엔드유저/API 채널 통합 집계, 인증 실패 별도 화면" 명시 검토

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md]] — 89-91번 줄 PM 확인 항목
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md]] — § 1.2 nginx log-watcher, § 4 P0 보강 리스트, § 5.3 api_call 사용자 매핑 한계
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] — H-DASH-01, H-DASH-03, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing
