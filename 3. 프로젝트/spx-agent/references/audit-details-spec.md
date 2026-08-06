---
tags: [프로젝트, dify, AI-Agent, references]
type: references
date: 2026-05-12
last_updated: 2026-05-15
purpose: 마트 ETL audit details 입력 필드 가용성 매트릭스 + P0 collector 보강 명세 + Generated Column 8 DDL
related: [audit-schema.md, dify-error-flow.md, defect-catalog.md#H-DASH-18]
verified_from: dify-audit/src/lib/collectors/*.ts (코드 직접 읽기)
---

# audit details 필드 가용성 매트릭스

## 변경 이력

- 2026-05-15: 5/15 결정 4건 반영 — Generated Column `_d` 접미사 컨벤션 / target_app_id UUID / is_debug에 rag-pipeline-debugging 추가 / collector app.mode JOIN 유지
- 2026-05-14: P0 collector 보강 명세 + Generated Column 8 DDL + 검증 시나리오 신설 (5/14 마트 설계 정식 이관)
- 2026-05-12: 초안 — 14 수집 경로 전수 점검 + 가용성 매트릭스

## 1. collector별 details 키 명세

> 소스: `dify-audit/src/lib/collectors/*.ts`, `dify-audit/scripts/setup_audit_triggers.sql`, `dify-audit/src/workers/log-watcher.ts`, `dify-audit/src/lib/self-audit.ts`
>
> collector 파일 총 16개 중: 유틸 2개(`db-collector.ts`, `save-event.ts`), 실 collector 11개, nginx log-watcher 1개, pg_trigger 1개(SQL), self-audit 1개 = **수집 경로 14종**.

### 1.1 DB 폴링 collector (source = `dify_db`, 5분 주기)

| # | collector 파일 | action(s) | details 키 목록 |
|---|---|---|---|
| 1 | `messages.ts` | `message_send` | `messageId`, `conversationId`, `query`, `answer`, `modelProvider`, `modelId`, `messageTokens`, `answerTokens`, `totalTokens`(computed), `responseLatency`, `totalPrice`, `currency`, `error`, `fromSource`, `invokeFrom`, `workflowRunId` |
| 2 | `workflow-runs.ts` | `workflow_execute` | `workflowRunId`, `workflowId`, `elapsedTime`, `totalTokens`, `error` |
| 3 | `workflow-nodes.ts` | `workflow_node_execute` | `nodeType`, `nodeId`, `title`, `workflowId`, `workflowRunId`, `appId`, `appName`, `elapsedTime`, `error`, `triggeredFrom` |
| 4 | `conversations.ts` | `conversation_start` | `conversationId`, `conversationName`, `invokeFrom`, `sessionId` |
| 5 | `app-changes.ts` | `app_create`, `app_update` | `mode`, `description`, `icon`, `iconType`, `createdAt` |
| 6 | `api-tokens.ts` | `api_token_create` | `tokenType`, `appId` |
| 7 | `datasets.ts` | `dataset_create` | `description`, `indexingTechnique` |
| 8 | `documents.ts` | `document_upload` | `datasetId`, `datasetName`, `indexingStatus`, `error`, `tokens`, `dataSourceType` |
| 9 | `members.ts` | `member_join`, `member_change` | `tenantId`, `accountEmail`, `role`, `joinId` |
| 10 | `message-feedbacks.ts` | `message_feedback` | `rating`, `feedbackContent`, `conversationId`, `messageId`, `appId`, `appName`, `fromSource` |
| 11 | `prompt-changes.ts` | `prompt_update` | `configId`, `promptPreview`(200자 truncate), `promptLength` |
| 12 | `provider-changes.ts` | `provider_model_add`, `provider_model_update` | `providerName`, `modelName`, `modelType`, `isValid`, `tenantId` |
| 13 | `workflow-publishes.ts` | `workflow_publish`, `workflow_draft` | `workflowId`, `appId`, `appName`, `type`, `version`, `markedName` |

### 1.2 nginx log-watcher (source = `nginx_log`, 실시간)

| action | details 키 목록 |
|---|---|
| `api_call` | `method`, `uri`, `httpStatus`, `requestTimeMs` |
| `auth_failed` | `method`, `uri`, `httpStatus`, `requestTimeMs` |
| `rate_limit_exceeded` | `method`, `uri`, `httpStatus`, `requestTimeMs` |

### 1.3 PostgreSQL trigger (source = `pg_trigger`, 실시간 DELETE/UPDATE)

| action | details 키 목록 |
|---|---|
| `document_delete` | `documentName`, `datasetId`, `dataSourceType`, `tokens` |
| `dataset_delete` | `datasetName`, `description`, `indexingTechnique` |
| `app_delete` | `appName`, `mode`, `description` |
| `api_token_delete` | `tokenType`, `appId`, `lastUsedAt` |
| `member_remove` | `role`, `tenantId` |
| `member_role_change` | `oldRole`, `newRole`, `tenantId` |

### 1.4 self-audit (source = `dify_audit_app`, 실시간)

| action | details 구조 |
|---|---|
| `audit_login`, `audit_logout`, `audit_login_denied` | caller가 자유 형태 전달 (보통 `{ provider }`) |
| `audit_list_view` | caller가 자유 형태 (보통 `{ totalResults, search, category, action }`) |
| `audit_detail_view` | caller가 자유 형태 |
| `audit_export` | caller가 자유 형태 (보통 `{ format, rowCount }`) |

### 1.5 system-logs collector (`system-logs.ts`)

> `system_logs` 테이블에 적재 (public schema 통일 후 `public.spx_system_logs` 예정) — `spx_audit_events`와 별개. 마트 ETL 대상 아님.

### 1.6 5/8 노트와의 차이

- 5/8 발견: "messages.ts 14필드 풍부 / workflow-runs.ts 5필드 비대칭" → **코드 재확인 결과 정확히 일치** (messages: 16키, workflow-runs: 5키)
- 5/8 시점 미점검 collector: 나머지 11종 + nginx/trigger/self-audit → **본 문서에서 전수 점검 완료**

---

## 2. 마트 ETL에 쓰이는 이벤트 action 범위

마트 ETL이 실제로 읽는 action은 **이벤트 차트** 관련 6종:

| action | collector | 용도 |
|---|---|---|
| `message_send` | messages.ts | 호출수, 토큰, 에러, 사용자 메트릭 |
| `workflow_execute` | workflow-runs.ts | 호출수, 토큰, 에러 |
| `workflow_node_execute` | workflow-nodes.ts | 노드 레벨 에러, triggeredFrom 필터 |
| `conversation_start` | conversations.ts | 사용자 세션 메트릭 |
| `api_call` | log-watcher (nginx) | API 호출 메트릭 |
| `auth_failed` | log-watcher (nginx) | 보안 에러 메트릭 |

나머지 action(admin/관리 이벤트)은 감사 로그 전용 — 마트 ETL 범위 밖.

---

## 3. 마트 ETL 입력 키 가용성 매트릭스

> **범례**: ✅ details에 존재 / 🔶 top-level 컬럼에 존재 (details 아님) / ❌ 없음 / ⚠️ 부분 가용

| ETL 키 | 용도 | `message_send` | `workflow_execute` | `workflow_node_execute` | `conversation_start` | `api_call`/`auth_failed` | 가용 판정 | 보강 필요 |
|---|---|:---:|:---:|:---:|:---:|:---:|---|---|
| **actorAccountId** | 호출자 계정 → RBAC JOIN | 🔶 `actorId` (actorType='account'일 때) | 🔶 `actorId` | 🔶 `actorId` | 🔶 `actorId` (actorType='account'일 때) | ❌ (actorType='api') | **가용** — top-level `actor_id` + `actor_type` 조합으로 추출. api_call은 IP만 있어 계정 매핑 불가 | 아니오 |
| **targetAppId** | 앱 → RBAC 부서 JOIN | 🔶 `targetId` | 🔶 `targetId` | ✅ `appId` (details) | 🔶 `targetId` | ❌ | **가용** — 대부분 top-level `target_id`. workflow_node는 targetType='workflow_node'라 details.appId 필요. api_call은 URI에서 앱 특정 불가 | 아니오 (api_call 제외) |
| **appMode** | AppMode 분기 (H-DASH-01) | ❌ | ❌ | ❌ | ❌ | ❌ | **❌ 완전 누락** — 어떤 이벤트 collector도 앱 모드를 details에 안 넣음. `app_create`/`app_update`에만 `mode` 존재하나 이벤트 action이 다름 | **🔴 필수 보강** |
| **modelProvider** | model-tokens 차트 | ✅ `modelProvider` | ❌ | ❌ | ❌ | ❌ | **부분** — message_send만. workflow는 구조적 한계 (H-DASH-02) | ⚠️ 선택 보강 |
| **modelId** | model-tokens 차트 | ✅ `modelId` | ❌ | ❌ | ❌ | ❌ | **부분** — message_send만 | ⚠️ 선택 보강 |
| **totalTokens** | 토큰 메트릭 | ✅ `totalTokens` | ✅ `totalTokens` | ❌ | ❌ | ❌ | **가용** — 두 핵심 action 모두 보유 | 아니오 |
| **error** | 에러 텍스트 (H-DASH-18) | ✅ `error` | ✅ `error` | ✅ `error` | ❌ | ❌ | **가용** — 3 action 모두 보유. 분류는 ILIKE만 (H-DASH-18) | 아니오 |
| **invokeFrom** | 디버깅 필터 (H-DASH-03) | ✅ `invokeFrom` | ❌ | ❌ | ✅ `invokeFrom` | ❌ | **❌ workflow_execute 누락** — messages.ts는 SELECT하지만, workflow-runs.ts는 `workflow_runs` 테이블의 관련 필드를 SELECT 안 함 | **🔴 필수 보강** |
| **triggeredFrom** | WF 트리거 분기 (H-DASH-03) | ❌ (해당 없음) | ❌ | ✅ `triggeredFrom` | ❌ | ❌ | **❌ workflow_execute 누락** — `workflow_runs.triggered_from` 컬럼이 Dify DB에 존재하나 collector가 SELECT 안 함. workflow_node_execute에만 있음 | **🔴 필수 보강** |
| **messageId** | 이벤트 식별 (드로어) | ✅ `messageId` | ❌ (해당 없음) | ❌ | ❌ | ❌ | **가용** — message_send에서만 필요 | 아니오 |
| **workflowRunId** | 이벤트 식별 (드로어) | ✅ `workflowRunId` | ✅ `workflowRunId` | ✅ `workflowRunId` | ❌ | ❌ | **가용** — 3 action 모두 보유 | 아니오 |
| **query/queryPreview** | 드로어 본문 표시 | ✅ `query` (전문) | ❌ | ❌ | ❌ | ❌ | **가용 (message_send)** — 단, 전문 저장이라 마트에는 preview(200자)로 truncate 권장. workflow는 입력 파라미터가 details에 없음 | ⚠️ 선택 보강 |
| **외부 사용자 식별** | actor가 end_user인지 구분 | 🔶 `actorType` = 'end_user' | 🔶 `actorType` = created_by_role | 🔶 `actorType` | 🔶 `actorType` = 'end_user' | 🔶 `actorType` = 'api' | **가용** — top-level `actor_type` 컬럼으로 구분 가능 | 아니오 |

### 3.1 요약 판정

| 상태 | 키 수 | 키 목록 |
|---|---|---|
| ✅ 가용 (보강 불필요) | 8 | actorAccountId, targetAppId, totalTokens, error, messageId, workflowRunId, query, 외부사용자 |
| 🔴 필수 보강 | 3 | **appMode**, **invokeFrom** (workflow), **triggeredFrom** (workflow) |
| ⚠️ 선택 보강 | 2 | modelProvider/modelId (workflow), queryPreview (workflow) |

---

## 4. collector 보강 요청 1차 리스트

### 🔴 P0 — 마트 ETL 차단 (보강 없으면 마트 불가)

| # | 누락 키 | 보강 대상 collector | 변경 내용 | 우선순위 |
|---|---|---|---|---|
| 1 | `appMode` | `messages.ts` | SQL에 `app.mode AS app_mode` 추가 → details에 `appMode: r.app_mode` | P0 |
| 2 | `appMode` | `workflow-runs.ts` | SQL에 `app.mode AS app_mode` 추가 → details에 `appMode: r.app_mode` | P0 |
| 3 | `appMode` | `workflow-nodes.ts` | SQL에 `app.mode AS app_mode` 추가 → details에 `appMode: r.app_mode` | P0 |
| 4 | `appMode` | `conversations.ts` | SQL에 `app.mode AS app_mode` 추가 → details에 `appMode: r.app_mode` | P0 |
| 5 | `triggeredFrom` | `workflow-runs.ts` | SQL에 `wr.triggered_from` 추가 → details에 `triggeredFrom: r.triggered_from` | P0 |
| 6 | `invokeFrom` | `workflow-runs.ts` | `workflow_runs` 테이블에 `invoke_from` 컬럼 유무 확인 필요. 없으면 `triggered_from`으로 대체 (debugging 판별: `triggered_from = 'debugging'`) | P0 |

> **참고**: `workflow_runs` 테이블의 디버깅 필터는 `triggered_from = 'debugging'`이므로 (H-DASH-03 참조), `triggeredFrom` 보강만으로 `invokeFrom` 대체 가능. 즉 #5 하나로 #5+#6 모두 커버.

### ⚠️ P1 — 있으면 좋지만 fallback 가능

| # | 누락 키 | 보강 대상 collector | 변경 내용 | fallback |
|---|---|---|---|---|
| 7 | `modelProvider`, `modelId` | `workflow-runs.ts` | `workflow_runs`에 해당 컬럼 없음 → 보강 불가. `workflow_node_executions.process_data` JSON 파싱 필요 | "미분류" 버킷 (H-DASH-02 기존 방어) |
| 8 | `queryPreview` | `workflow-runs.ts` | 워크플로우 입력 파라미터가 `workflow_runs.inputs` JSON에 있으나 구조가 앱마다 다름 | 드로어에서 OLTP `workflow_runs.inputs` 직접 조회 |

### 보강 작업량 추정

- P0 (#1~#6): collector `.ts` 파일 4개 수정, 각 파일에 SQL 컬럼 1~2개 + details 키 1~2개 추가. **총 ~30줄 변경**. 난이도 낮음.
- P1 (#7~#8): 구조적 한계로 collector 보강으로는 해결 안 됨 → 마트 ETL에서 fallback 처리.

---

## 5. 외부 사용자 처리 확인

### 5.1 actor_type으로 구분

audit_events의 top-level `actor_type` 컬럼이 사용자 유형을 구분:

| actor_type | 의미 | 설정 위치 |
|---|---|---|
| `account` | Dify 워크스페이스 멤버 (관리자/편집자) | `messages.ts`: `from_account_id` 존재 시 |
| `end_user` | 외부 앱 사용자 | `messages.ts`: `from_account_id` 없을 때 |
| `api` | nginx API 호출 (토큰 기반, 계정 불명) | `log-watcher.ts`: 항상 `api` |
| `system` | 자동 작업 | `provider-changes.ts` |

### 5.2 end_user 세부 식별

- `actor_id`에 `end_users.id` (Dify 내부 UUID) 저장됨
- `conversations.ts`만 `sessionId` (end_user의 session_id) 추가 제공
- end_user의 이름/외부ID는 details에 **없음** → 필요 시 OLTP `end_users` 테이블 JOIN 필요
- **마트 ETL 관점**: 부서 기준 = 앱 소유 부서이므로 end_user 세부 식별은 필수 아님. 활성 사용자 수 카운트만 필요하면 `actor_id` COUNT DISTINCT로 충분

### 5.3 api_call의 사용자 매핑 한계

- nginx log에서 수집하는 `api_call`/`auth_failed`는 HTTP 레벨이라 **계정 정보 없음**
- `actor_type = 'api'`, IP + User-Agent만 보유
- 마트에서 이 이벤트를 부서별로 집계하려면 **URI에서 앱 토큰 → 앱 → 부서** 역추적이 필요하나, URI만으로는 앱 특정 불가
- **결론**: api_call은 전체 호출량/에러율 메트릭으로만 활용. 부서별 분류 불가 (보강으로도 해결 어려움 — nginx가 bearer token을 로그에 안 남김)

---

## 6. Harness 후보 (본 검증에서 발견)

### H-CAND-audit-appmode-missing

| 항목 | 내용 |
|---|---|
| **가설** | audit collector 4종(messages, workflow-runs, workflow-nodes, conversations)이 `app.mode`를 details에 안 넣어서, audit 기반 마트에서 H-DASH-01 AppMode 분기 방어가 불가능 |
| **확정 근거** | 코드 직접 읽기로 확인 — 4개 collector SQL + details 객체에 mode 키 없음 |
| **승격 판단** | 코드 소스로 확정됨 → collector 보강 전까지 **마트 ETL 차단 요소** |

### H-CAND-audit-wf-debug-filter-missing

| 항목 | 내용 |
|---|---|
| **가설** | `workflow-runs.ts` collector가 `triggered_from` 컬럼을 SELECT 안 하고 details에도 안 넣어서, audit 기반 마트에서 H-DASH-03 디버깅 필터 방어가 불가능 |
| **확정 근거** | 코드 직접 읽기로 확인 — `workflow-runs.ts` SQL에 `triggered_from` 없음 |
| **승격 판단** | 코드 소스로 확정됨 → collector 보강 전까지 **마트 ETL 차단 요소** |

### H-CAND-audit-query-fulltext

| 항목 | 내용 |
|---|---|
| **관찰** | `messages.ts`가 `query` 전문(full text)을 details에 저장. 사용자 질문 전체가 audit DB에 들어감 |
| **리스크** | (a) audit_events 테이블 용량 급증 (query 길이 수천자 가능) (b) 개인정보 포함 가능성 (사용자가 개인정보를 질문에 입력) |
| **임시 판단** | 마트 ETL에서는 `queryPreview` (200자 truncate)만 사용 권장. collector 보강 시 `queryPreview: r.query?.slice(0, 200)` 별도 키 추가 고려 |

---

## P0 collector 보강 명세 (2026-05-14 마트 설계 § 7 기반)

> 마트 풀가동 차단 요소. § 4 P0 1차 리스트의 정식 명세본. 사용자 본인 트랙으로 진행 (5/14 우선순위 2).

### 보강 대상 4 파일 × 추가 키 매트릭스

| collector | 추가 키 | SQL 추가 컬럼 | 영향 Generated Column | 방어 Harness | 사용처 (마트 객체) |
|---|---|---|---|---|---|
| `messages.ts` | `appMode` | `app.mode AS app_mode` | app_mode | H-DASH-01 | v_audit_enriched `is_canonical_call` 분기 (advanced-chat 이중카운트 방어) |
| `workflow-runs.ts` | `appMode` | `app.mode AS app_mode` | app_mode | H-DASH-01 | v_audit_enriched `is_canonical_call` (workflow_execute가 advanced-chat 소속이면 제외) |
| `workflow-runs.ts` | `triggeredFrom` | `wr.triggered_from` | triggered_from | H-DASH-03 | v_audit_enriched `is_debug` 분기 (workflow_runs의 디버깅 필터 = `triggered_from='debugging'`) |
| `workflow-nodes.ts` | `appMode` | `app.mode AS app_mode` | app_mode | H-DASH-01 | (현재 마트 범위 밖, 미래 노드 레벨 분석용 선제 보강) |
| `conversations.ts` | `appMode` | `app.mode AS app_mode` | app_mode | H-DASH-01 | (현재 마트 범위 밖, 세션 메트릭 도입 시 분기용) |

> § 4 표의 `invokeFrom` (workflow) 항목은 별도 보강 불필요 — `workflow_runs.triggered_from` 보강만으로 `triggered_from = 'debugging'` 판별 가능 (H-DASH-03 코드 표준). 즉 `triggeredFrom` 추가 한 줄로 invoke/triggered 디버깅 필터 모두 커버.

### 작업 윤곽

- 보강 작업량: 4 collector 파일 × SQL 컬럼 1~2개 + details 키 1~2개 = **총 ~30줄 변경**
- 난이도: 낮음 (기존 SQL에 컬럼 1개 추가 + JS 객체 키 1개 추가)
- 검증 흐름: P0 보강 → 신규 INSERT만 Generated Column 채움 → 백필 무필요 (audit_events 데이터 0건이라 영향 없음, 운영 진입 시점부터 자연 충족)

### 정확한 patch diff

> 출처: [[0. Inbox/collector 보강 + Generated Column 선행 분석 - 2026-05-14.md]] § A.1 ~ A.5 (VSCode Claude Code 코드 직접 분석)
>
> 5/15 결정: `app.mode` LEFT JOIN 유지 — `messages.app_mode` 직접 사용 안 함 (레거시 행 호환 안전 선택). conversations.ts는 `c.mode` 직접 사용 가능 (NOT NULL).

#### A.1 `messages.ts` — `appMode` 추가

> 파일: `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\messages.ts`
> 위치: row 타입 line 33, SQL line 56~57, details line 95~97

```diff
--- a/src/lib/collectors/messages.ts
+++ b/src/lib/collectors/messages.ts
@@ -33,6 +33,7 @@
     workflow_run_id: string | null
     created_at: Date
     tenant_id: string | null
+    app_mode: string | null
   }>>`
     SELECT
       m.id::text AS id,
@@ -56,7 +57,8 @@
       m.invoke_from,
       m.workflow_run_id::text AS workflow_run_id,
       m.created_at,
-      app.tenant_id::text AS tenant_id
+      app.tenant_id::text AS tenant_id,
+      app.mode AS app_mode
     FROM messages m
     LEFT JOIN apps app ON m.app_id = app.id
     LEFT JOIN accounts a ON m.from_account_id = a.id
@@ -95,6 +97,7 @@
       fromSource: r.from_source,
       invokeFrom: r.invoke_from,
       workflowRunId: r.workflow_run_id,
+      appMode: r.app_mode,
     },
     source: 'dify_db',
     tenantId: r.tenant_id,
```

#### A.2 `workflow-runs.ts` — `appMode` + `triggeredFrom` 추가

> 파일: `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\workflow-runs.ts`
> 위치: row 타입 line 22, SQL line 36~39, details line 62~66

```diff
--- a/src/lib/collectors/workflow-runs.ts
+++ b/src/lib/collectors/workflow-runs.ts
@@ -22,6 +22,8 @@
     actor_email: string | null
     error: string | null
     tenant_id: string | null
+    app_mode: string | null
+    triggered_from: string
   }>>`
     SELECT
       wr.id::text as id,
@@ -36,7 +38,9 @@
       a.email as actor_email,
       wr.error,
-      app.tenant_id::text as tenant_id
+      app.tenant_id::text as tenant_id,
+      app.mode as app_mode,
+      wr.triggered_from
     FROM workflow_runs wr
     LEFT JOIN apps app ON wr.app_id = app.id
     LEFT JOIN accounts a ON (wr.created_by_role = 'account' AND wr.created_by = a.id)
@@ -62,6 +66,8 @@
       elapsedTime: r.elapsed_time,
       totalTokens: r.total_tokens,
       error: r.error,
+      appMode: r.app_mode,
+      triggeredFrom: r.triggered_from,
     },
     source: 'dify_db',
     tenantId: r.tenant_id,
```

#### A.3 `workflow-nodes.ts` — `appMode` 추가

> 파일: `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\workflow-nodes.ts`
> 위치: row 타입 line 28, SQL line 45~48, details line 78~80

```diff
--- a/src/lib/collectors/workflow-nodes.ts
+++ b/src/lib/collectors/workflow-nodes.ts
@@ -28,6 +28,7 @@
     actor_email: string | null
     triggered_from: string
     tenant_id: string | null
+    app_mode: string | null
   }>>`
     SELECT
       n.id::text AS id,
@@ -45,7 +46,8 @@
       n.created_by_role,
       a.email AS actor_email,
       n.triggered_from,
-      app.tenant_id::text AS tenant_id
+      app.tenant_id::text AS tenant_id,
+      app.mode AS app_mode
     FROM workflow_node_executions n
     LEFT JOIN apps app ON n.app_id = app.id
     LEFT JOIN accounts a ON (n.created_by_role = 'account' AND n.created_by = a.id)
@@ -78,6 +80,7 @@
       elapsedTime: r.elapsed_time,
       error: r.error,
       triggeredFrom: r.triggered_from,
+      appMode: r.app_mode,
     },
     source: 'dify_db',
     tenantId: r.tenant_id,
```

#### A.4 `conversations.ts` — `appMode` 추가 (c.mode 직접 사용)

> 파일: `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\conversations.ts`
> 위치: row 타입 line 19, SQL line 29~30, details line 60~62
>
> `conversations.mode` 컬럼이 NOT NULL이고 생성 시 `app.mode` 스냅샷이 박혀 있어 직접 사용 가능. 마트 목적상 호출 시점 mode가 필요하므로 스냅샷 OK.

```diff
--- a/src/lib/collectors/conversations.ts
+++ b/src/lib/collectors/conversations.ts
@@ -19,6 +19,7 @@
     name: string | null
     created_at: Date
     invoke_from: string | null
+    app_mode: string
     tenant_id: string | null
   }>>`
     SELECT
@@ -29,6 +30,7 @@
       a.email as account_email,
       c.name,
       c.created_at,
       c.invoke_from,
+      c.mode AS app_mode,
       app.tenant_id::text as tenant_id
     FROM conversations c
     LEFT JOIN apps app ON c.app_id = app.id
@@ -60,6 +62,7 @@
       conversationName: r.name,
       invokeFrom: r.invoke_from,
       sessionId: r.end_user_session_id,
+      appMode: r.app_mode,
     },
     source: 'dify_db',
     tenantId: r.tenant_id,
```

#### A.5 Collector 분석 요약

| collector | 추가 키 | 소스 컬럼 | JOIN 추가? | 변경 라인 수 |
|---|---|---|---|---|
| messages.ts | `appMode` | `app.mode` | 불필요 (기존 LEFT JOIN apps) | +3 |
| workflow-runs.ts | `appMode`, `triggeredFrom` | `app.mode`, `wr.triggered_from` | 불필요 (기존 LEFT JOIN apps) | +5 |
| workflow-nodes.ts | `appMode` | `app.mode` | 불필요 (기존 LEFT JOIN apps) | +3 |
| conversations.ts | `appMode` | `c.mode` (직접 컬럼) | 불필요 | +3 |
| **합계** | | | **JOIN 추가 0건** | **~14줄** |

> 4 collector 모두 이미 `apps` LEFT JOIN 보유 → SELECT 컬럼만 추가하면 끝. `conversations.ts`는 `c.mode` 직접 사용 (NOT NULL 컬럼).

---

## Generated Column 8개 DDL (2026-05-14 마트 설계 § 4.1)

> 마트 적용 시 `public.spx_audit_events`에 일괄 ALTER. 데이터 0건이라 백필 lock 비용 0 (운영 트래픽 영향 없음).
> ※ **선행**: `audit_events` 테이블을 `spx_audit_events`로 rename + audit schema → public schema 이전 (dify-audit collector INSERT target 동반 수정). 본 ALTER은 rename 완료 후 실행.

-- 5/15 결정: _d 접미사 컨벤션(details 추출 표시), total_tokens_d BIGINT, target_app_id UUID(resource_ownership.resource_id JOIN 직접)
```sql
ALTER TABLE public.spx_audit_events
  ADD COLUMN app_mode_d        TEXT   GENERATED ALWAYS AS (details->>'appMode')         STORED,
  ADD COLUMN model_provider_d  TEXT   GENERATED ALWAYS AS (details->>'modelProvider')   STORED,
  ADD COLUMN model_id_d        TEXT   GENERATED ALWAYS AS (details->>'modelId')         STORED,
  ADD COLUMN total_tokens_d    BIGINT GENERATED ALWAYS AS (NULLIF(details->>'totalTokens','')::bigint) STORED,
  ADD COLUMN error_d           TEXT   GENERATED ALWAYS AS (details->>'error')           STORED,
  ADD COLUMN invoke_from_d     TEXT   GENERATED ALWAYS AS (details->>'invokeFrom')      STORED,
  ADD COLUMN triggered_from_d  TEXT   GENERATED ALWAYS AS (details->>'triggeredFrom')   STORED,
  ADD COLUMN target_app_id     UUID   GENERATED ALWAYS AS (
    CASE
      WHEN target_type = 'app' THEN target_id::uuid
      WHEN details ? 'appId'   THEN (details->>'appId')::uuid
      ELSE NULL
    END
  ) STORED;
```

### 컬럼별 출처 키 + collector 보강 의존

| 컬럼 | 출처 키 | 채워지는 action | collector 보강 의존 |
|---|---|---|---|
| `app_mode_d` | `details->>'appMode'` | message_send, workflow_execute, workflow_node_execute, conversation_start | **🔴 P0 4건** (messages, workflow-runs, workflow-nodes, conversations) |
| `model_provider_d` | `details->>'modelProvider'` | message_send만 | ✅ 기존 보강 (messages.ts) |
| `model_id_d` | `details->>'modelId'` | message_send만 | ✅ 기존 보강 |
| `total_tokens_d` | `details->>'totalTokens'` | message_send, workflow_execute | ✅ 기존 보강. 5/15 타입 BIGINT (workflow_runs.total_tokens=bigint 오버플로 방지) |
| `error_d` | `details->>'error'` | message_send, workflow_execute, workflow_node_execute, document_upload | ✅ 기존 보강. 5/15: 컬럼명 `error_text` → `error_d` (`_d` 접미사 통일) |
| `invoke_from_d` | `details->>'invokeFrom'` | message_send, conversation_start | ✅ 기존 보강 (workflow_execute는 `triggered_from_d`로 대체) |
| `triggered_from_d` | `details->>'triggeredFrom'` | workflow_node_execute (기존), workflow_execute (**P0 보강 후**) | **🔴 P0 workflow-runs.ts** |
| `target_app_id` | top-level `target_id` (target_type='app') / `details->>'appId'` (workflow_node_execute) | 모든 action | ✅ 기존 데이터로 충족. 5/15 타입 UUID (resource_ownership.resource_id JOIN 직접). 예외: `_d` 접미사 미부여 — top-level + details 혼용이라 details 추출 컨벤션 미적용 |

### 마이그레이션 절차 (Prisma Migrate)

**도구**: Prisma Migrate (Alembic 아님). 마이그레이션 폴더: `dify-audit/prisma/audit/migrations/`

**선행 조건**:
- PostgreSQL 15.15 (PG 12+ STORED Generated Column 지원 ✅)
- public schema CREATE 권한 (postgres superuser ✅)
- `audit_events` → `spx_audit_events` rename + audit schema → public 이전 **선행 완료** 필수 (5/15 우선순위 2 트랙)

**Generated Column은 Prisma ORM이 직접 미지원 → raw SQL migration 패턴**:

1. `pnpm prisma migrate dev --create-only --name add_generated_columns_for_mart` → 빈 migration 폴더 생성
2. 자동생성된 `migration.sql`에 위 § Generated Column 8개 DDL의 ALTER TABLE 블록 수동 작성
3. 아래 § 인덱스 5개 CREATE INDEX 추가 (`CONCURRENTLY` 옵션은 트랜잭션 밖이라 migration에서는 사용 불가 → 운영 적용 시 별도 SQL로 수동 실행 권장)
4. `pnpm prisma migrate dev` 적용 + `pnpm prisma migrate status` 검증

**downgrade**: `DROP COLUMN target_app_id, triggered_from_d, invoke_from_d, error_d, total_tokens_d, model_id_d, model_provider_d, app_mode_d` (역순)

---

### 인덱스 5개 (마트 enriched REFRESH 가속용)

> 출처: [[0. Inbox/collector 보강 + Generated Column 선행 분석 - 2026-05-14.md]] § F.2

```sql
-- 마트 enriched REFRESH 가속용 인덱스 5개
CREATE INDEX spx_audit_events_action_mart_idx
  ON public.spx_audit_events (action)
  WHERE action IN ('message_send','workflow_execute');  -- partial, enriched 메인 필터

CREATE INDEX spx_audit_events_tenant_target_app_idx
  ON public.spx_audit_events (tenant_id, target_app_id);  -- ro JOIN

CREATE INDEX spx_audit_events_tenant_actor_idx
  ON public.spx_audit_events (tenant_id, actor_id);  -- dept_members JOIN

CREATE INDEX spx_audit_events_occurred_action_idx
  ON public.spx_audit_events (occurred_at, action);  -- 미래 incremental refresh

CREATE INDEX spx_audit_events_app_mode_idx
  ON public.spx_audit_events (app_mode_d);  -- is_canonical_call 분기
```

---

## 검증 시나리오 (2026-05-14 마트 설계 § 8 통합 시나리오 축약)

P0 보강 + Generated Column 적용 후 4 collector가 정상 작동하는지 검증.

### 시나리오: 4 collector 모두 트리거하는 가짜 호출 1세트

1. **messages.ts 트리거**: advanced-chat 앱에서 채팅 메시지 1건 전송 → `public.spx_audit_events`에 `action='message_send'` 1행
2. **workflow-runs.ts 트리거**: workflow 앱에서 워크플로우 1회 실행 → `action='workflow_execute'` 1행
3. **workflow-nodes.ts 트리거**: 위 워크플로우 안에 LLM 노드 1개 → `action='workflow_node_execute'` 1행
4. **conversations.ts 트리거**: advanced-chat 앱에서 새 대화 시작 → `action='conversation_start'` 1행

### 검증 쿼리: 새 키 + Generated Column 자동 계산 확인

```sql
-- 4 collector 모두 appMode가 details에 들어왔는가
SELECT action, details->>'appMode' AS detail_app_mode, app_mode_d
FROM public.spx_audit_events
WHERE occurred_at >= NOW() - INTERVAL '10 minutes'
  AND action IN ('message_send','workflow_execute','workflow_node_execute','conversation_start')
ORDER BY occurred_at DESC;

-- 기대: detail_app_mode = app_mode_d (NULL이 아닌 값), 4행 모두 채워짐
```

```sql
-- workflow_execute의 triggered_from 보강 확인
SELECT details->>'triggeredFrom' AS detail_tf, triggered_from_d
FROM public.spx_audit_events
WHERE action='workflow_execute' AND occurred_at >= NOW() - INTERVAL '10 minutes';

-- 기대: detail_tf = triggered_from_d (보통 'app-run', 디버깅이면 'debugging' 또는 'rag-pipeline-debugging')
```

```sql
-- v_audit_enriched의 is_canonical_call / is_debug 플래그가 정확히 분기되는가
-- enriched view 내부에서 app_mode_d → app_mode로 alias됨 (마트 컴포넌트 쿼리 단순화)
SELECT action, app_mode, is_canonical_call, is_debug
FROM public.spx_mv_audit_enriched
WHERE occurred_at >= NOW() - INTERVAL '10 minutes'
ORDER BY occurred_at DESC;

-- 기대:
--   message_send + advanced-chat → is_canonical_call=TRUE
--   workflow_execute + advanced-chat → is_canonical_call=FALSE (H-DASH-01 방어)
--   triggered_from_d='app-run' → is_debug=FALSE
--   triggered_from_d IN ('debugging','rag-pipeline-debugging') → is_debug=TRUE (5/15 결정)
```

→ 위 쿼리들이 모두 기대대로 나오면 마트 풀가동 가능. 메트릭 #3 (enriched vs src 행수 차이) 0이면 ownership 매핑까지 정상.

---

## 7. top-level vs details 키 정리 (ETL 설계 참고)

마트 ETL은 `details->>` JSONB 추출 외에 top-level 컬럼도 활용해야 함:

| ETL에서 쓰는 정보 | 추출 경로 | 비고 |
|---|---|---|
| 호출자 계정 ID | `actor_id` (top-level) + `actor_type = 'account'` 필터 | |
| 앱 ID | `target_id` (top-level, targetType='app'일 때) 또는 `details->>'appId'` (workflow_node_execute) | action별 분기 필요 |
| 앱 모드 | **현재 없음** → 보강 후 `details->>'appMode'` | P0 보강 |
| 모델 정보 | `details->>'modelProvider'`, `details->>'modelId'` | message_send만 |
| 토큰 | `details->>'totalTokens'` | message_send + workflow_execute |
| 에러 | `details->>'error'` | |
| 디버깅 필터 | message_send: `details->>'invokeFrom' != 'debugger'` / workflow_execute: `details->>'triggeredFrom' NOT IN ('debugging','rag-pipeline-debugging')` | 5/15 결정: `rag-pipeline-debugging` 추가 (`WorkflowRunTriggeredFrom` enum 7종 중 디버깅 의미 2종) |
| 이벤트 ID | `details->>'messageId'` 또는 `details->>'workflowRunId'` | |
| 발생 시각 | `occurred_at` (top-level) | 기간 필터용 |
| 테넌트 | `tenant_id` (top-level) | 멀티테넌트 격리 |
