---
tags: [프로젝트, dify, AI-Agent, 분석, audit]
date: 2026-05-14
purpose: P0 collector 보강 + audit_events Generated Column 8개 마이그레이션 선행 분석
related:
  - "[[3. 프로젝트/spx-agent/references/audit-details-spec.md]]"
  - "[[3. 프로젝트/spx-agent/references/audit-schema.md]]"
  - "[[3. 프로젝트/spx-agent/references/dify-db-schema.md]]"
  - "[[0. Inbox/마트 설계 결정 - 2026-05-14.md]]"
---

# collector 보강 + Generated Column 마이그레이션 선행 분석

> 코드 수정 없음. 분석/보고만.

---

## A. Collector 4파일 분석

### A.1 messages.ts

| 항목                     | 내용                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **파일 경로**              | `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\messages.ts`                                                                                                                                                                                                                                                                                                                                                                                                    |
| **소스 테이블**             | `messages m` LEFT JOIN `apps app` ON `m.app_id = app.id` LEFT JOIN `accounts a` ON `m.from_account_id = a.id`                                                                                                                                                                                                                                                                                                                                                                            |
| **폴링 메커니즘**            | `getCursor('messages')` → `collector_state.cursor_value` (ISO timestamp). `WHERE m.created_at > ${since}` 증분 폴링. LIMIT 5000. 5분 cron 주기                                                                                                                                                                                                                                                                                                                                                  |
| **현재 details 키 (16개)** | `messageId`(m.id), `conversationId`(m.conversation_id), `query`(m.query), `answer`(m.answer), `modelProvider`(m.model_provider), `modelId`(m.model_id), `messageTokens`(m.message_tokens), `answerTokens`(m.answer_tokens), `totalTokens`(computed: message+answer), `responseLatency`(m.provider_response_latency), `totalPrice`(m.total_price), `currency`(m.currency), `error`(m.error), `fromSource`(m.from_source), `invokeFrom`(m.invoke_from), `workflowRunId`(m.workflow_run_id) |
| **`appMode` 가져오는 방법**  | `messages.app_mode` 컬럼 존재 — Dify 코드가 **항상 세팅함** (`api/core/app/apps/message_based_app_generator.py:216`, `app_mode=app_config.app_mode`). 2025-10-14 마이그레이션(`d98acf217d43`)으로 추가된 컬럼. 단, **마이그레이션 이전에 생성된 messages는 NULL/빈 문자열**일 수 있음. 안전 선택으로 `app.mode` JOIN 사용 권장 (LEFT JOIN apps 이미 존재). `m.app_mode` 직접 사용도 가능하나 레거시 행 호환 불확실 |
| **추가 INSERT 위치**       | SQL: line 57 `app.tenant_id::text AS tenant_id` 뒤에 `app.mode AS app_mode` 추가. TypeScript row 타입: line 34 `tenant_id` 뒤에 `app_mode: string \| null` 추가. details: line 97 `workflowRunId` 뒤에 `appMode: r.app_mode` 추가                                                                                                                                                                                                                                                                      |

**패치 diff 초안:**

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

---

### A.2 workflow-runs.ts

| 항목 | 내용 |
|---|---|
| **파일 경로** | `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\workflow-runs.ts` |
| **소스 테이블** | `workflow_runs wr` LEFT JOIN `apps app` ON `wr.app_id = app.id` LEFT JOIN `accounts a` ON `(wr.created_by_role = 'account' AND wr.created_by = a.id)` |
| **폴링 메커니즘** | 동일 (getCursor, timestamp 증분, LIMIT 5000, 5분 cron) |
| **현재 details 키 (5개)** | `workflowRunId`(wr.id), `workflowId`(wr.workflow_id), `elapsedTime`(wr.elapsed_time), `totalTokens`(wr.total_tokens), `error`(wr.error) |
| **`appMode` 가져오는 방법** | `workflow_runs`에 mode/app_mode 컬럼 **없음**. → `app.mode` 사용 (LEFT JOIN apps 이미 존재) |
| **`triggeredFrom` / `invokeFrom`** | `wr.triggered_from` 컬럼 존재 (varchar NOT NULL). Dify `WorkflowRunTriggeredFrom` enum (`api/models/enums.py:22-29`): **7종** — `debugging`, `app-run`, `rag-pipeline-run`, `rag-pipeline-debugging`, `webhook`, `schedule`, `plugin`. `invoke_from` 컬럼은 **없음**. `triggered_from`만 보강하면 H-DASH-03 방어 충분 (마트: `triggered_from = 'debugging'` 필터. `rag-pipeline-debugging`도 디버깅이므로 필터 표현식 재검토 필요) |
| **추가 INSERT 위치** | SQL: line 39 `app.tenant_id::text as tenant_id` 뒤. TypeScript: line 24 `tenant_id` 뒤. details: line 65 `error` 뒤 |

**패치 diff 초안:**

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

---

### A.3 workflow-nodes.ts

| 항목 | 내용 |
|---|---|
| **파일 경로** | `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\workflow-nodes.ts` |
| **소스 테이블** | `workflow_node_executions n` LEFT JOIN `apps app` ON `n.app_id = app.id` LEFT JOIN `accounts a` ON `(n.created_by_role = 'account' AND n.created_by = a.id)` |
| **폴링 메커니즘** | 동일 (getCursor, timestamp 증분, LIMIT 5000, 5분 cron) |
| **현재 details 키 (10개)** | `nodeType`, `nodeId`, `title`, `workflowId`, `workflowRunId`, `appId`, `appName`, `elapsedTime`, `error`, `triggeredFrom` |
| **`appMode` 가져오는 방법** | `workflow_node_executions`에 mode/app_mode 컬럼 **없음**. → `app.mode` 사용 (LEFT JOIN apps 이미 존재) |
| **추가 INSERT 위치** | SQL: line 48 `app.tenant_id::text AS tenant_id` 뒤. TypeScript: line 30 `tenant_id` 뒤. details: line 80 `triggeredFrom` 뒤 |

**패치 diff 초안:**

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

---

### A.4 conversations.ts

| 항목 | 내용 |
|---|---|
| **파일 경로** | `C:\Users\Administrator\Projects\spx-agent\dify-audit\src\lib\collectors\conversations.ts` |
| **소스 테이블** | `conversations c` LEFT JOIN `apps app` ON `c.app_id = app.id` LEFT JOIN `end_users eu` ON `c.from_end_user_id = eu.id` LEFT JOIN `accounts a` ON `c.from_account_id = a.id` |
| **폴링 메커니즘** | 동일 (getCursor, timestamp 증분, LIMIT 5000, 5분 cron) |
| **현재 details 키 (4개)** | `conversationId`(c.id), `conversationName`(c.name), `invokeFrom`(c.invoke_from), `sessionId`(eu.session_id) |
| **`appMode` 가져오는 방법** | ✅ `conversations.mode` 컬럼 직접 존재 (NOT NULL). Dify 코드가 conversation 생성 시 **`app.mode` 스냅샷으로 세팅** — `message_based_app_generator.py:173`에서 `mode=app_config.app_mode.value`, `workflow_draft_variable_service.py:608`에서 `mode=app.mode`. **stale 가능성**: app.mode 변경 시 기존 conversation.mode는 자동 갱신 안 됨 (수동 배치 `system.py:108-116`만 존재). 마트 목적상 **호출 시점 mode가 필요하므로 스냅샷 OK** — `c.mode` 직접 사용 가능 |
| **추가 INSERT 위치** | SQL: line 30 `c.invoke_from` 뒤에 `c.mode AS app_mode` 추가 (또는 `app.mode` 사용도 가능). TypeScript: line 21 `invoke_from` 뒤. details: line 63 `sessionId` 뒤 |

**패치 diff 초안:**

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

---

### A.5 Collector 분석 요약

| collector | 추가 키 | 소스 컬럼 | JOIN 추가? | 변경 라인 수 |
|---|---|---|---|---|
| messages.ts | `appMode` | `app.mode` | 불필요 (기존 JOIN) | +3 |
| workflow-runs.ts | `appMode`, `triggeredFrom` | `app.mode`, `wr.triggered_from` | 불필요 (기존 JOIN) | +5 |
| workflow-nodes.ts | `appMode` | `app.mode` | 불필요 (기존 JOIN) | +3 |
| conversations.ts | `appMode` | `c.mode` (직접 컬럼!) | 불필요 | +3 |
| **합계** | | | **JOIN 추가 0건** | **~14줄** |

> **핵심 발견**: 4 collector 모두 이미 `apps` 테이블을 LEFT JOIN하고 있어서 SQL JOIN 추가 없이 SELECT 컬럼만 추가하면 됨. `conversations.ts`는 자체 `mode` 컬럼도 가용.

---

## B. Dify 소스 테이블 스키마 확인 (코드 기준)

> 근거: Dify upstream `api/models/model.py`, `api/models/enums.py`, `api/core/app/apps/message_based_app_generator.py` 코드 직접 확인.

### B.1 컬럼 실재 여부

| 테이블 | 컬럼 | 타입 | 실재? | 코드 근거 |
|---|---|---|---|---|
| messages | `app_mode` | `EnumText(AppMode)`, nullable | ✅ | `model.py:1432`. Dify가 INSERT 시 **항상 세팅** (`message_based_app_generator.py:216`). 단 2025-10-14 마이그레이션(`d98acf217d43`) 이전 레거시 행은 NULL/빈 문자열 가능 → 안전 선택으로 `app.mode` JOIN 사용 |
| messages | `invoke_from` | varchar, nullable | ✅ | `InvokeFrom` enum 7종 (§ B.3) |
| workflow_runs | `triggered_from` | varchar, NOT NULL | ✅ | `WorkflowRunTriggeredFrom` enum 7종 (§ B.3) |
| workflow_runs | `invoke_from` | — | ❌ **없음** | model 정의에 해당 컬럼 없음. `triggered_from`으로 대체 |
| workflow_runs | `app_mode` / `mode` | — | ❌ **없음** | `apps` JOIN 필요 |
| workflow_node_executions | `triggered_from` | varchar, NOT NULL | ✅ | collector가 이미 SELECT 중 |
| workflow_node_executions | `app_mode` / `mode` | — | ❌ **없음** | `apps` JOIN 필요 |
| conversations | `mode` | `EnumText(AppMode)`, NOT NULL | ✅ | `model.py:1051`. 생성 시 `app.mode` 스냅샷 (§ A.4 코드 근거 참조). 직접 사용 가능 |
| conversations | `invoke_from` | varchar, nullable | ✅ | collector가 이미 SELECT 중 |
| apps | `mode` | `EnumText(AppMode)`, NOT NULL | ✅ | `model.py` — 모든 AppMode enum 값 가능 |

### B.2 AppMode enum 정의 (코드 기준)

소스: `api/models/model.py:349-356` — `class AppMode(StrEnum)`

| Enum 멤버 | 값 | 마트 관련 | 비고 |
|---|---|---|---|
| COMPLETION | `completion` | ✅ | 프롬프트 1회 입력 → 결과 |
| CHAT | `chat` | ✅ | 기본 채팅 |
| AGENT_CHAT | `agent-chat` | ✅ | 도구 사용 에이전트 |
| ADVANCED_CHAT | `advanced-chat` | ✅ | 워크플로우 + 채팅 UI. **H-DASH-01 핵심** |
| WORKFLOW | `workflow` | ✅ | 워크플로우 캔버스 1회성 |
| CHANNEL | `channel` | ❌ | 코드에 enum만 존재, 실제 앱 생성 경로 없음 |
| RAG_PIPELINE | `rag-pipeline` | ❌ | 코드에 enum만 존재, 실제 앱 생성 경로 없음 |

> `dify-app-modes.md`에 5종 명시 — 맞으나, **코드 enum은 7종**. 마트 `app_mode TEXT`로 모든 값 수용 가능. `is_canonical_call` 플래그는 `advanced-chat` 비교만 하므로 추가 enum 값에 영향 없음.

### B.3 InvokeFrom / WorkflowRunTriggeredFrom enum (코드 기준)

**`InvokeFrom`** — `api/models/enums.py:178-187` — `class InvokeFrom(StrEnum)`:

| 값 | 마트 필터 | 비고 |
|---|---|---|
| `service-api` | 포함 | 외부 API 호출 |
| `web-app` | 포함 | 웹앱 사용자 |
| `trigger` | 포함 | 자동 트리거 |
| `explore` | 포함 | 탐색 페이지 |
| `debugger` | **제외 (H-DASH-03)** | Studio 디버깅 |
| `published` | 포함 | 게시된 파이프라인 |
| `validation` | 포함 | 검증 호출 |

**`WorkflowRunTriggeredFrom`** — `api/models/enums.py:22-29` — `class WorkflowRunTriggeredFrom(StrEnum)`:

| 값 | 마트 필터 | 비고 |
|---|---|---|
| `app-run` | 포함 | 정상 실행 |
| `debugging` | **제외 (H-DASH-03)** | Studio 디버깅 |
| `rag-pipeline-run` | 포함 | RAG 파이프라인 실행 |
| `rag-pipeline-debugging` | **제외 가능** | RAG 디버깅 — ⚠️ 마트 필터 표현식에 포함 여부 결정 필요 |
| `webhook` | 포함 | 웹훅 트리거 |
| `schedule` | 포함 | 스케줄 트리거 |
| `plugin` | 포함 | 플러그인 트리거 |

> ⚠️ **마트 설계 영향**: 현재 enriched `is_debug` = `triggered_from = 'debugging'`만 체크. `rag-pipeline-debugging`도 디버깅이므로 **`triggered_from IN ('debugging', 'rag-pipeline-debugging')` 또는 `LIKE '%debugging%'`으로 확장 필요**. 사용자 결정 대기.

### B.4 `messages.app_mode` 컬럼 분석 (코드 기준)

**Dify 코드가 app_mode를 항상 세팅하는 것을 확인:**

- **Model 정의**: `api/models/model.py:1432` — `app_mode: Mapped[AppMode | None] = mapped_column(EnumText(AppMode, length=255), nullable=True)`
- **세팅 위치**: `api/core/app/apps/message_based_app_generator.py:216` — `Message(... app_mode=app_config.app_mode ...)`
- **유일한 Message 생성 경로**: `_save_message()` 메서드가 모든 Message INSERT의 단일 진입점
- **마이그레이션**: `2025_10_14_1618-d98acf217d43_add_app_mode_for_messsage.py` — 이 시점 이전 행은 NULL

**collector 패치 판정**: `m.app_mode`을 직접 사용하면 레거시 행(마이그레이션 이전)에서 NULL/빈 문자열 위험. `app.mode` JOIN이 안전 선택. 다만 운영 환경에서 레거시 행이 없다면(= 시스템이 2025-10 이후 구축) `m.app_mode` 직접 사용도 가능. **현재 패치 diff는 `app.mode` JOIN 유지 — 레거시 호환 안전.**

---

## C. 마이그레이션 컨벤션

### C.1 ORM/마이그레이션 도구

| 항목 | 내용 |
|---|---|
| ORM | **Prisma** (Prisma Client JS) |
| 마이그레이션 도구 | **Prisma Migrate** (Alembic 아님) |
| 마이그레이션 폴더 | `dify-audit/prisma/audit/migrations/` |
| 스키마 파일 | `dify-audit/prisma/audit/schema.prisma` |
| DB 스키마 | `audit` (Prisma `@@map` + SQL `CREATE SCHEMA IF NOT EXISTS "audit"`) |

### C.2 기존 마이그레이션 패턴

| 마이그레이션 | revision | 패턴 |
|---|---|---|
| `20260428022406_init_audit` | timestamp 기반 자동생성 | `CREATE TABLE`, `CREATE INDEX` — 순수 DDL SQL |
| `20260430000000_add_tenant_id_and_system_logs` | 수동 timestamp (`000000`) | `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`, `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS` — 멱등 패턴 |

- **audit schema 명시 방법**: `ALTER TABLE "audit"."audit_events"` (schema-qualified 테이블명)
- **Generated Column**: Prisma ORM 자체는 Generated Column을 **직접 지원하지 않음**. 방법:
  1. `prisma migrate dev --create-only` → 빈 migration 폴더 생성 → `migration.sql`에 raw SQL 수동 작성
  2. 또는 `prisma db execute --file add_generated_columns.sql` 직접 실행
  3. **권장**: 방법 1 (마이그레이션 이력에 남음)

### C.3 PostgreSQL 버전

```
PostgreSQL 15.15 on x86_64-pc-linux-musl, compiled by gcc (Alpine 15.2.0) 15.2.0, 64-bit
```

✅ **PG 15 — `GENERATED ALWAYS AS ... STORED` 완전 지원** (PG 12+). 차단 요소 없음.

### C.4 권한

```
audit schema ACL: {postgres=UC/postgres, audit_writer=UC/postgres}
현재 세션: postgres (superuser)
```

✅ `postgres` superuser로 `ALTER TABLE` 권한 충분. `audit_writer`도 UC(Usage+Create) 보유.

---

## D. Generated Column 타입 매핑 엣지 케이스

### D.1 기존 데이터 현황

> 우리 환경에 audit_events 데이터가 있음 (mock fixture 등). 운영 환경 데이터 양은 별개.
> ALTER TABLE 시 기존 행에 대해 Generated Column 표현식이 평가되므로, **표현식이 기존 details 구조와 호환되는지**가 핵심. details에 해당 키가 없으면 NULL 반환 — 안전.

### D.2 컬럼별 검토 (collector 코드 기준)

> 근거: dify-audit `src/lib/collectors/*.ts` 코드의 TypeScript 타입 정의 + details 객체 구성 직접 확인.

| 컬럼 | 표현식 | collector 코드 근거 | 판정 | 권장 처리 |
|---|---|---|---|---|
| `app_mode` | `details->>'appMode'` | 보강 후 4 collector가 `appMode: r.app_mode`으로 박음 (string). 보강 전 기존 행은 키 없음 → NULL | ✅ OK | 없음 |
| `model_provider` | `details->>'modelProvider'` | `messages.ts:86` — `modelProvider: r.model_provider` (string \| null). messages만 보유, 나머지 action은 키 없음 → NULL | ✅ OK | 없음 |
| `model_id` | `details->>'modelId'` | `messages.ts:87` — `modelId: r.model_id` (string \| null). 동일 | ✅ OK | 없음 |
| `total_tokens` | `NULLIF(...)::int` | `messages.ts:90` — `totalTokens: r.message_tokens + r.answer_tokens` (number 연산 결과, null 불가). `workflow-runs.ts:65` — `totalTokens: r.total_tokens` (number \| **null**). JSONB에 number로 저장 → `->>`는 text "1394" 반환 → `::int` 정상. **BUT** Dify `workflow_runs.total_tokens` = bigint → 극단 값 시 int 오버플로 | ⚠️ **BIGINT** | `::bigint` 변경 |
| `error_text` | `details->>'error'` | 4 collector 모두 `error: r.error` (string \| null). collector 코드에서 빈 문자열 변환 없음 — null이면 JSONB `"error": null` → `->>`는 SQL NULL 반환 | ✅ OK | 없음 |
| `invoke_from` | `details->>'invokeFrom'` | `messages.ts:96` — `invokeFrom: r.invoke_from` (string \| null). `conversations.ts:61` — 동일. 나머지 action은 키 없음 → NULL | ✅ OK | 없음 |
| `triggered_from` | `details->>'triggeredFrom'` | `workflow-nodes.ts:80` — `triggeredFrom: r.triggered_from` (string, NOT null). 보강 후 `workflow-runs.ts`에도 추가. 나머지 NULL | ✅ OK | 없음 |
| `target_app_id` | CASE 표현식 | 아래 상세 | ⚠️ 주의 | TEXT 타입 권장 |

### D.3 `target_app_id` CASE 표현식 상세 분석

설계 표현식:
```sql
CASE
  WHEN target_type = 'app' THEN target_id::uuid
  WHEN details ? 'appId'   THEN (details->>'appId')::uuid
  ELSE NULL
END
```

**코드 기준 분석 — collector별 targetType/targetId/details.appId:**

| collector | targetType | targetId 출처 | UUID 보장 | details에 `appId` | 비고 |
|---|---|---|---|---|---|
| messages.ts | `app` | `m.app_id::text` (Dify UUID) | ✅ | ❌ | CASE 첫 번째 분기 |
| workflow-runs.ts | `app` | `wr.app_id::text` (Dify UUID) | ✅ | ❌ | CASE 첫 번째 분기 |
| conversations.ts | `app` | `c.app_id::text` (Dify UUID) | ✅ | ❌ | CASE 첫 번째 분기 |
| app-changes.ts | `app` | `app.id::text` (Dify UUID) | ✅ | ❌ | 〃 |
| prompt-changes.ts | `app` | `app.id::text` (Dify UUID) | ✅ | ❌ | 〃 |
| workflow-nodes.ts | `workflow_node` | `n.id::text` (Dify UUID) | ✅ | ✅ `r.app_id` (UUID) | CASE 두 번째 분기. `appId` = `apps.id::text` |
| workflow-publishes.ts | `workflow` | `w.id::text` (Dify UUID) | ✅ | ✅ `r.app_id` (UUID) | CASE 두 번째 분기 |
| api-tokens.ts | `api_token` | `t.id::text` (Dify UUID) | ✅ | ✅ `r.app_id` (nullable UUID) | CASE 두 번째, NULL 가능 → ELSE NULL |
| message-feedbacks.ts | `message` | `m.id::text` (Dify UUID) | ✅ | ✅ `r.app_id` (UUID) | CASE 두 번째 분기 |
| datasets.ts | `dataset` | `d.id::text` (Dify UUID) | ✅ | ❌ | ELSE NULL |
| documents.ts | `document` | `d.id::text` (Dify UUID) | ✅ | ❌ | ELSE NULL |
| members.ts | `account` | `a.id::text` (Dify UUID) | ✅ | ❌ | ELSE NULL |
| provider-changes.ts | `provider_model` | `pm.id::text` (Dify UUID) | ✅ | ❌ | ELSE NULL |
| nginx log-watcher | `endpoint` | NULL | — | ❌ | ELSE NULL |

**코드 기준 판정**:
- **모든 collector의 `targetId`는 Dify DB UUID 컬럼의 `::text` 캐스트** — 코드상 비-UUID 값이 들어갈 경로 없음
- **`details.appId`를 박는 4 collector** 모두 `apps.id::text` (UUID) — 코드상 비-UUID 불가
- 그러나 **향후 collector 추가/수정 시 비-UUID 값이 실수로 들어올 가능성**은 배제 못 함

**권장**: `target_app_id`를 **TEXT 타입**으로 선언 — UUID 캐스팅 실패 위험 원천 제거 + `resource_ownership.resource_id`(TEXT)와 직접 비교 가능

```sql
-- TEXT 타입 권장안
CASE
  WHEN target_type = 'app' THEN target_id
  WHEN details ? 'appId'   THEN details->>'appId'
  ELSE NULL
END
```

### D.4 수정 권장 DDL

```sql
ALTER TABLE audit.audit_events
  ADD COLUMN app_mode        TEXT    GENERATED ALWAYS AS (details->>'appMode')         STORED,
  ADD COLUMN model_provider  TEXT    GENERATED ALWAYS AS (details->>'modelProvider')   STORED,
  ADD COLUMN model_id        TEXT    GENERATED ALWAYS AS (details->>'modelId')         STORED,
  ADD COLUMN total_tokens    BIGINT  GENERATED ALWAYS AS (NULLIF(details->>'totalTokens','')::bigint) STORED,
  ADD COLUMN error_text      TEXT    GENERATED ALWAYS AS (details->>'error')           STORED,
  ADD COLUMN invoke_from_d   TEXT    GENERATED ALWAYS AS (details->>'invokeFrom')      STORED,
  ADD COLUMN triggered_from_d TEXT   GENERATED ALWAYS AS (details->>'triggeredFrom')   STORED,
  ADD COLUMN target_app_id   TEXT    GENERATED ALWAYS AS (
    CASE
      WHEN target_type = 'app' THEN target_id
      WHEN details ? 'appId'   THEN details->>'appId'
      ELSE NULL
    END
  ) STORED;
```

**변경점 vs 5/14 마트 설계:**

| 컬럼 | 변경 | 사유 |
|---|---|---|
| `total_tokens` | `INT` → `BIGINT` | workflow_runs.total_tokens가 bigint. 오버플로 방지 |
| `invoke_from_d` | 이름에 `_d` 접미사 | audit_events 기존 설계에 top-level `invoke_from` 컬럼 추가 가능성과 이름 충돌 방지. **또는** 마트 설계에서 Generated Column 이름을 `detail_invoke_from` 등으로 명확화 |
| `triggered_from_d` | 이름에 `_d` 접미사 | 동일 |
| `target_app_id` | `UUID` → `TEXT` | UUID 캐스팅 실패 시 INSERT reject 방지. resource_ownership.resource_id(TEXT)와 직접 비교 가능 |

> ⚠️ **이름 충돌 주의**: audit_events에 이미 top-level 컬럼이 없으므로 `invoke_from` / `triggered_from` 그대로 사용 가능하긴 함. 단 미래에 Prisma schema에 해당 컬럼이 추가되면 충돌. **사용자 결정 필요**.

---

## E. 검증 시나리오 설계

### E.1 트리거 시나리오 (실 앱 호출)

collector가 5분 주기로 가동 중이므로 **실 앱 호출로 검증 가능**.

| # | 시나리오 | 트리거 방법 | 검증 collector | 예상 action |
|---|---|---|---|---|
| 1 | chat 앱 메시지 1건 | Dify Studio → chat 앱 → "hello" 전송 | messages.ts | `message_send` |
| 2 | workflow 앱 1회 실행 | Dify Studio → workflow 앱 → "Run" 클릭 | workflow-runs.ts + workflow-nodes.ts | `workflow_execute` + `workflow_node_execute` ×N |
| 3 | advanced-chat 메시지 1건 | Dify Studio → advanced-chat 앱 → "hello" 전송 | messages.ts (H-DASH-01 검증) | `message_send` (advanced-chat도 messages만) |
| 4 | 새 대화 시작 | 시나리오 1 자동 수반 (첫 메시지 = conversation_start) | conversations.ts | `conversation_start` |

> 5분 cron 대기 후 audit_events에 적재 확인.

### E.2 Mock INSERT 대안

collector 수정 전 Generated Column만 먼저 테스트하려면 mock INSERT:

```sql
-- collector 보강된 details 형태 모의
INSERT INTO audit.audit_events (
  id, occurred_at, category, action,
  actor_type, actor_id, target_type, target_id, target_name,
  status, details, source, tenant_id
) VALUES (
  'test_gc_' || gen_random_uuid()::text,
  NOW(), 'user', 'message_send',
  'account', '559f9254-4c3d-489d-9815-1f05ad03a011',
  'app', 'e1b90001-0000-0000-0000-000000000001', 'Test Chat App',
  'success',
  '{
    "messageId": "test-msg-001",
    "appMode": "chat",
    "modelProvider": "anthropic",
    "modelId": "claude-sonnet-4-20250514",
    "totalTokens": 1500,
    "invokeFrom": "web-app",
    "error": null
  }'::jsonb,
  'manual',
  'test-tenant-001'
);

-- workflow_execute mock (triggeredFrom 보강)
INSERT INTO audit.audit_events (
  id, occurred_at, category, action,
  actor_type, actor_id, target_type, target_id, target_name,
  status, details, source, tenant_id
) VALUES (
  'test_gc_' || gen_random_uuid()::text,
  NOW(), 'user', 'workflow_execute',
  'account', '559f9254-4c3d-489d-9815-1f05ad03a011',
  'app', 'e1b90001-0000-0000-0000-000000000002', 'Test Workflow App',
  'succeeded',
  '{
    "workflowRunId": "test-wr-001",
    "workflowId": "test-wf-001",
    "appMode": "workflow",
    "triggeredFrom": "app-run",
    "totalTokens": 3200,
    "error": null
  }'::jsonb,
  'manual',
  'test-tenant-001'
);
```

### E.3 검증 쿼리

```sql
-- Generated Column 자동 계산 확인
SELECT
  action,
  app_mode,            -- Generated: details->>'appMode'
  model_provider,      -- Generated: details->>'modelProvider'
  total_tokens,        -- Generated: NULLIF(...)::bigint
  invoke_from_d,       -- Generated: details->>'invokeFrom'
  triggered_from_d,    -- Generated: details->>'triggeredFrom'
  target_app_id,       -- Generated: CASE ...
  error_text,          -- Generated: details->>'error'
  -- 원본과 대조
  details->>'appMode'       AS raw_app_mode,
  details->>'invokeFrom'    AS raw_invoke_from,
  occurred_at
FROM audit.audit_events
WHERE id LIKE 'test_gc_%'
ORDER BY occurred_at DESC;
```

```sql
-- 정리
DELETE FROM audit.audit_events WHERE id LIKE 'test_gc_%';
```

---

## F. 인덱스 5개 후보

### F.1 마트 쿼리 패턴 추출

enriched REFRESH 쿼리 (가장 중요 — 5분마다 전체 audit_events 스캔):
```sql
FROM audit.audit_events ae
WHERE ae.action IN ('message_send', 'workflow_execute')
-- JOIN on (ae.tenant_id, ae.target_app_id) → resource_ownership
-- JOIN on (ae.tenant_id, ae.actor_id) → department_members
```

### F.2 인덱스 후보 5개

| # | 컬럼 | 인덱스 타입 | 적용 근거 |
|---|---|---|---|
| **1** | `(action) WHERE action IN ('message_send','workflow_execute')` | **Partial B-tree** | enriched REFRESH의 메인 필터. collector 14종 중 마트 대상 action은 2종뿐 (`message_send`, `workflow_execute`). 나머지 12종(auth_failed, app_create, dataset_create, document_upload 등)은 감사 로그 전용 → partial index로 마트 대상 행만 빠르게 필터. **가장 효과 큰 인덱스** |
| **2** | `(tenant_id, target_app_id)` | Composite B-tree (Generated Column) | enriched의 `INNER JOIN resource_ownership ON (tenant_id, resource_id=target_app_id)`. merge/hash join build 지원 |
| **3** | `(tenant_id, actor_id)` | Composite B-tree | enriched의 `LEFT JOIN department_members ON (tenant_id, account_id=actor_id)`. 현재도 `actor_id` 단독 인덱스 있으나 tenant 복합이 더 선택적 |
| **4** | `(occurred_at, action)` | Composite B-tree | 향후 incremental refresh 전환 시 (`WHERE occurred_at > last_refresh AND action IN (...)`) 핵심. 현재 full refresh에서도 sort 지원 |
| **5** | `(app_mode)` (Generated Column) | B-tree | `is_canonical_call` 플래그 계산 (`app_mode <> 'advanced-chat' OR action='message_send'`). 직접 인덱스보다 enriched MView에서 `is_canonical_call` boolean 인덱스가 더 효과적일 수 있음 — **대안 검토 가능** |

### F.3 기존 인덱스와 중복 체크

현재 audit_events 인덱스 (init_audit migration):
```
audit_events_occurred_at_idx       → (occurred_at)       — 유지
audit_events_category_idx          → (category)          — 마트 미사용, 감사 로그 UI용
audit_events_action_idx            → (action)            — #1 partial index가 대체 (기존 유지해도 OK)
audit_events_actor_email_idx       → (actor_email)       — 감사 로그 UI용
audit_events_actor_id_idx          → (actor_id)          — #3이 tenant 복합으로 확장
audit_events_target_type_target_id → (target_type, target_id) — 감사 로그 UI용
audit_events_tenant_id_idx         → (tenant_id)         — #2, #3의 leading 컬럼
```

> 후보 #1(partial)과 #2(target_app_id)는 완전 신규. #3은 기존 actor_id 인덱스 확장. #4는 기존 occurred_at 인덱스와 leading 동일하나 action 추가. #5는 Generated Column 신설 후 가능.

### F.4 MView 자체 인덱스 (CONCURRENTLY 전제)

`REFRESH MATERIALIZED VIEW CONCURRENTLY`를 사용하려면 각 MView에 **UNIQUE INDEX** 필수:

```sql
-- v_audit_enriched
CREATE UNIQUE INDEX ON mart.v_audit_enriched (id);

-- mv_kpi_calls_daily
CREATE UNIQUE INDEX ON mart.mv_kpi_calls_daily (tenant_id, day, COALESCE(app_owner_dept_id,''), COALESCE(actor_dept_id,''), target_app_id, app_mode);

-- mv_model_tokens_daily
CREATE UNIQUE INDEX ON mart.mv_model_tokens_daily (tenant_id, day, model_provider, model_id);
```

Layer 2 MView 조회 인덱스:

```sql
-- kpi-cards / dept-activity 기간 조회
CREATE INDEX ON mart.mv_kpi_calls_daily (tenant_id, day);

-- model-tokens 기간 조회
CREATE INDEX ON mart.mv_model_tokens_daily (tenant_id, day);
```

---

## 결론 요약

### 차단 요소

| # | 항목 | 차단? | 상태 |
|---|---|---|---|
| 1 | PostgreSQL 버전 (STORED 지원) | ❌ 없음 | PG 15.15 ✅ |
| 2 | audit schema ALTER 권한 | ❌ 없음 | postgres superuser ✅ |
| 3 | 기존 데이터 호환 | ❌ 없음 | Generated Column 표현식은 details에 키 없으면 NULL — 안전 |
| 4 | Prisma Generated Column 지원 | ⚠️ **부분** | raw SQL migration으로 우회 (Prisma 공식 패턴) |
| 5 | `messages.app_mode` 레거시 호환 | ⚠️ **주의** | Dify 코드는 항상 세팅 (2025-10 마이그레이션 이후). 패치는 안전 선택으로 `app.mode` JOIN 유지 |
| 6 | `target_app_id` UUID 캐스팅 실패 | ⚠️ **주의** | 코드상 모든 targetId/appId가 UUID지만, TEXT 타입이 안전 (§ D.3) |
| 7 | `invoke_from` / `triggered_from` 이름 충돌 | ⚠️ **결정 필요** | 접미사 `_d` 사용 vs 그대로 — 사용자 결정 |
| 8 | `total_tokens` INT 오버플로 | ⚠️ **수정 필요** | BIGINT으로 변경 (§ D.4) |
| 9 | `rag-pipeline-debugging` 디버깅 필터 | ⚠️ **결정 필요** | `WorkflowRunTriggeredFrom` 7종 중 디버깅 2종 존재. 마트 `is_debug` 표현식 확장 여부 (§ B.3) |

### 사용자 결정 대기 항목

1. **Generated Column 이름**: `invoke_from` / `triggered_from` 그대로 vs `invoke_from_d` / `triggered_from_d`
2. **`target_app_id` 타입**: TEXT(안전, 권장) vs UUID(정규식 가드 필요)
3. **디버깅 필터 확장**: `is_debug` = `triggered_from IN ('debugging', 'rag-pipeline-debugging')` vs `= 'debugging'`만. `rag-pipeline-debugging`이 마트 집계에서 제외되어야 하는지 판정 필요
4. **`messages.app_mode` 직접 사용 여부**: 운영 환경이 2025-10 이후 구축이라면 `m.app_mode` 직접 사용 가능 (JOIN 제거로 collector SQL 단순화). 현재 패치는 안전 선택 `app.mode` JOIN 유지

### 결론

**차단 요소 없음. P0 패치 명세 작성 준비 완료.**

collector 4파일 변경 ~14줄 + Generated Column ALTER TABLE 1개 migration + 인덱스 5개 = 총 작업량 소규모. Prisma raw SQL migration 패턴으로 수행 가능.

### 이전 버전 대비 변경 사항 (코드 기반 재조사)

| 항목 | 이전 판단 (데이터 기반) | 수정 판단 (코드 기반) |
|---|---|---|
| `messages.app_mode` | "Dify가 채우지 않음, 빈 문자열 함정" | **Dify가 항상 세팅** (`message_based_app_generator.py:216`). 2025-10 마이그레이션 이전 레거시만 주의 |
| `conversations.mode` 일치 검증 | "apps.mode와 완전 일치 검증 완료" (DB 조인) | 코드 확인: **app.mode 스냅샷** (생성 시 복사, 자동 갱신 X). `system.py:108-116`에 수동 배치만 존재 |
| `WorkflowRunTriggeredFrom` | "2종 (`app-run` / `debugging`)" | **7종** — `rag-pipeline-debugging`도 디버깅 → 필터 확장 결정 필요 |
| `InvokeFrom` | "`web-app`, `debugger`, `app-run` 등" | **7종** 전체 확인 — `service-api`, `web-app`, `trigger`, `explore`, `debugger`, `published`, `validation` |
| `AppMode` enum | "5종 + CHANNEL/RAG_PIPELINE 비활성" | 코드 enum **7종** 확인 (`model.py:349-356`). dify-app-modes.md 범위와 일치하나 코드 근거로 교체 |
| audit_events 데이터 | "658건, action 분포 표" | 데이터 분포 표 삭제 — 운영 분포 미지. 코드상 collector 14종 action 화이트리스트로 대체 |
| target_id UUID 보장 | "데이터 검증: 모든 값 유효 UUID" | **코드 검증**: 13 collector 모두 Dify UUID 컬럼 `::text` 캐스트. 코드상 비-UUID 경로 없음 |
| details 값 타입 | "실 데이터에서 확인" | **collector TypeScript 타입 정의 기준**: totalTokens=number\|null, error=string\|null, modelProvider=string\|null 등 |
