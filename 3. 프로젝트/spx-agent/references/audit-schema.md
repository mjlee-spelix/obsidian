---
tags: [프로젝트, dify, AI-Agent, references]
type: references
last_updated: 2026-05-15
purpose: Audit DB 스키마 — `public` schema의 `spx_` prefix 5종 테이블 + Generated Column 8개 + 인덱스 + trigger
related: [audit-details-spec.md, hdd/design.md#2.5]
---

## 변경 이력

- 2026-05-15: rename 적용 (audit→public+spx_) + Generated Column 8개 + 인덱스 5개 신설 + 5/15 결정 4건 반영 (컬럼 `_d` 접미사 / `target_app_id` UUID / `total_tokens_d` BIGINT / trigger 함수 `public.spx_log_dify_change()`)
- 2026-05-14: 명명 변경 결정 (audit schema → public + spx_ prefix). rename 작업은 2026-05-15에 실행
- 초기: dify-audit 셀프호스트 audit schema 5종 테이블 명세

# Audit DB 스키마

dify-audit이 사용하는 `public` schema의 `spx_` prefix 테이블 5종. 모두 Dify Postgres의 `dify` 데이터베이스 안에 있음.

> **2026-05-15 rename 완료**: 기존 `audit` schema → `public` schema 이전 + `spx_` prefix 부여 완료. `audit` schema는 `DROP SCHEMA ... CASCADE`로 폐기. dify-audit collector INSERT target 동반 수정 완료. 변경 매핑은 `hdd/design.md § 2.5.0` 참조.

## 1. `public.spx_audit_events`

이벤트 기록 메인 테이블 (구조화된 감사 이벤트).

### 1.1 기본 컬럼 (top-level, 17개)

| 컬럼             | 타입           | 설명                                                                   |
| -------------- | ------------ | -------------------------------------------------------------------- |
| `id`           | text PK      | cuid                                                                 |
| `occurred_at`  | timestamp(3) | 이벤트 발생 시각                                                            |
| `category`     | text         | `admin` / `user` / `security`                                        |
| `action`       | text         | `prompt_update` / `api_call` / `app_delete` 등                        |
| `actor_type`   | text         | `account` / `end_user` / `api` / `system`                            |
| `actor_id`     | text         | actor_type별 형식 다름 (아래 표) — 마트 캐스트 시 H-MART-01 가드 필수      |
| `actor_email`  | text         | enrichment용 (대부분 NULL)                                               |
| `target_type`  | text         | `app` / `dataset` / `endpoint` 등                                     |
| `target_id`    | text         |                                                                      |
| `target_name`  | text         |                                                                      |
| `ip_address`   | text         |                                                                      |
| `user_agent`   | text         |                                                                      |
| `status`       | text         | `success` / `failed` / `running` 등                                   |
| `details`      | jsonb        | 액션별 가변 메타                                                            |
| `source`       | text         | `dify_db` / `nginx_log` / `dify_audit_app` / `pg_trigger` / `manual` |
| `collected_at` | timestamp(3) | 적재 시각                                                                |
| `tenant_id`    | text         | 워크스페이스 ID, NULL = 글로벌                                                |

### 1.1.1 `actor_id` 형식 — actor_type별 (2026-05-19 추가)

> **5/18 운영 안전 보강 학습 반영** (H-MART-01). 단순 `actor_id::uuid` 캐스트가 깨질 잠복 risk.

| actor_type | actor_id 형식 | 예시 | 마트 캐스트 시 처리 |
|---|---|---|---|
| `account` | **UUID** (Dify `spx_accounts.id`) | `559f9254-4c3d-489d-9815-1f05ad03a011` | `actor_id::uuid` 안전. `department_members.account_id`와 직접 JOIN 가능 |
| `end_user` | **UUID** (Dify `end_users.id`) | `e0000000-0000-0000-0000-000000000042` | `actor_id::uuid` 안전. 단 부서 매핑 없음(외부 사용자) — LEFT JOIN miss → sentinel UUID |
| `api` | **문자열** (비-UUID) | `'chatbot-public-001'`, `'public-api-key-xyz'` | **`::uuid` 캐스트 금지** — `invalid input syntax` 폭발. enriched view에서 `actor_id`는 NULL로 매핑 |
| `system` | **문자열** (비-UUID, 고정) | `'system'` | **`::uuid` 캐스트 금지** — 동일 폭발. enriched view에서 NULL 매핑 |

**마트 캐스트 표준 패턴** (5/18 마이그레이션 `20260518100000_actor_id_non_uuid_safe`):
```sql
-- SELECT 단계: 캐스트 가드
CASE WHEN actor_type IN ('account','end_user') THEN actor_id::uuid ELSE NULL END AS actor_id

-- JOIN 단계: account 한정 매칭
LEFT JOIN public.department_members dm
       ON dm.tenant_id = ae.tenant_id::uuid
      AND dm.account_id = CASE WHEN ae.actor_type='account' THEN ae.actor_id::uuid END
      AND dm.is_active = TRUE
```

**잠복 risk 사고 예방**: mock fixture 작성 시 `actor_type` 4종 모두 포함 의무 (5/18 1차 drift 정리 시점엔 mock에 account/end_user만 있어 검증 통과 → 운영 진입 전 미연 차단). 관련 가드: H-MART-01.

### 1.2 Generated Column 8개 (2026-05-15 추가, 마트용)

> 상세 명세: [[3. 프로젝트/spx-agent/references/audit-details-spec.md]] § Generated Column 8개 DDL. 컨벤션: details에서 추출한 컬럼은 `_d` 접미사 (target_app_id 예외).

| 컬럼 | 타입 | 표현식 | 사용처 |
|---|---|---|---|
| `app_mode_d` | TEXT | `details->>'appMode'` | `is_canonical_call` (H-DASH-01 분기) |
| `model_provider_d` | TEXT | `details->>'modelProvider'` | mv_model_tokens_daily group key |
| `model_id_d` | TEXT | `details->>'modelId'` | mv_model_tokens_daily group key |
| `total_tokens_d` | **BIGINT** | `NULLIF(details->>'totalTokens','')::bigint` | mv_model_tokens_daily SUM (workflow_runs.total_tokens=bigint 오버플로 방지) |
| `error_d` | TEXT | `details->>'error'` | 에러 분류 (H-DASH-18 ILIKE 입력) |
| `invoke_from_d` | TEXT | `details->>'invokeFrom'` | `is_debug` 분기 (message_send 경로) |
| `triggered_from_d` | TEXT | `details->>'triggeredFrom'` | `is_debug` 분기 (workflow_execute 경로). `IN ('debugging','rag-pipeline-debugging')` 필터 |
| `target_app_id` | **UUID** | `CASE WHEN target_type='app' THEN target_id::uuid WHEN details ? 'appId' THEN (details->>'appId')::uuid ELSE NULL END` | resource_ownership.resource_id 직접 JOIN. `_d` 접미사 예외 — top-level + details 혼용 |

### 1.3 인덱스 (총 13개)

**기존 8개** (top-level 컬럼):
- `spx_audit_events_pkey`
- `spx_audit_events_occurred_at_idx`
- `spx_audit_events_category_idx`
- `spx_audit_events_action_idx`
- `spx_audit_events_actor_email_idx`
- `spx_audit_events_actor_id_idx`
- `spx_audit_events_target_type_target_id_idx`
- `spx_audit_events_tenant_id_idx`

**마트 가속용 5개** (2026-05-15 추가):
- `spx_audit_events_action_mart_idx` — partial: `WHERE action IN ('message_send','workflow_execute')`. enriched 메인 필터
- `spx_audit_events_tenant_target_app_idx` — `(tenant_id, target_app_id)`. ro JOIN
- `spx_audit_events_tenant_actor_idx` — `(tenant_id, actor_id)`. dept_members JOIN
- `spx_audit_events_occurred_action_idx` — `(occurred_at, action)`. 미래 incremental refresh
- `spx_audit_events_app_mode_idx` — `(app_mode_d)`. is_canonical_call 분기

## 2. `public.spx_collector_state`

13개 폴링 collector의 진행 상태.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | text PK | cuid |
| `collector_name` | text UNIQUE | `app_changes` / `workflow_runs` 등 |
| `last_run_at` | timestamp(3) | 마지막 폴링 시각 |
| `cursor_value` | text | ISO timestamp 또는 ID |
| `meta` | jsonb | 옵션 메타 |
| `updated_at` | timestamp(3) | auto-updated |

## 3. `public.spx_log_file_state`

nginx access.log watcher의 파일 offset.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | text PK | cuid |
| `file_path` | text UNIQUE | 로그 파일 경로 |
| `last_read_at` | timestamp(3) | 마지막 읽기 시각 |
| `last_offset` | bigint | 어디까지 읽었나 (바이트) |
| `updated_at` | timestamp(3) | auto-updated |

## 4. `public.spx_system_logs`

일일 배치로 적재한 컨테이너 stdout 원시 로그 + spx_audit_events 텍스트 스냅샷.

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | text PK | cuid |
| `occurred_at` | timestamp(3) | 로그 발생 시각 |
| `source` | text | 컨테이너명 (예: `docker-api-1`) 또는 `audit_event` |
| `log_level` | text | `INFO` / `WARN` / `ERROR` / `DEBUG` / NULL |
| `message` | text | 본문 (8000자 trim) |
| `raw` | text | 원본 라인 (16000자 trim) |
| `batch_date` | date | 어느 일자 배치 |
| `ingested_at` | timestamp(3) | 적재 시각 |

**인덱스**: `occurred_at`, `source`, `log_level`, `batch_date`

## 5. `public.spx_system_log_batch`

일일 배치 진행 추적 (재실행 방지).

| 컬럼 | 타입 | 설명 |
|---|---|---|
| `id` | text PK | cuid |
| `source` | text | 컨테이너명 또는 `audit_event` |
| `batch_date` | date | 처리 대상 날짜 |
| `started_at` | timestamp(3) | |
| `finished_at` | timestamp(3) | NULL이면 진행중/실패 |
| `status` | text | `running` / `completed` / `failed` |
| `line_count` | integer | 적재된 라인 수 |
| `error_message` | text | 실패 사유 |

**유니크**: `(source, batch_date)`

---

## 6. 운영 시 주의사항 (2026-05-15 rename 사고 학습)

### Trigger 함수 + Trigger 5종

- **함수**: `public.spx_log_dify_change()` (구 `audit.log_dify_change()`)
- **Trigger 5종** (Dify 본체 테이블에 부착, DELETE/UPDATE 시 spx_audit_events INSERT):
  - `trg_audit_documents_delete` — `public.documents`
  - `trg_audit_datasets_delete` — `public.datasets`
  - `trg_audit_apps_delete` — `public.apps`
  - `trg_audit_api_tokens_delete` — `public.api_tokens`
  - `trg_audit_tenant_joins_change` — `public.tenant_account_joins` (DELETE + UPDATE 양쪽)
- **schema rename 시 trigger 재배포 필수** — pg_trigger는 함수 schema가 바뀌면 자동 갱신 안 됨

### Schema 권한 (audit_writer role)

- `audit_writer`는 `SELECT/INSERT/UPDATE/DELETE/REFERENCES/TRIGGER/TRUNCATE`만 보유
- **`ALTER` 권한 없음** → 마이그레이션 적용 시 `prisma migrate deploy` 실패 가능
- 운영 적용 시 두 옵션:
  - (a) 마이그레이션 직전에 `audit_writer`에게 임시 ALTER 권한 GRANT 후 회수
  - (b) postgres superuser로 직접 DDL 실행 + `prisma migrate resolve --applied`로 정합 처리 (5/15 사고 수습 시 채택)

### `setup_audit_schema.sql` 재발 방지

- `DROP SCHEMA IF EXISTS audit;` → `DROP SCHEMA IF EXISTS audit CASCADE;` (5/15 사고 재발 방지 주석 박음)
- 비-CASCADE로는 종속 객체(테이블/trigger/함수)가 남아 있어 silent fail. 다음 rename 시도 시 같은 사고 재현 가능

### log-watcher 옛 테이블 INSERT 사고

- 5/15 rename 직후 log-watcher가 옛 `audit.audit_events` 17건 INSERT 시도 → generated Prisma client 시점 이슈로 추정 (해소됨)
- 향후 rename 시점에 dify-audit 서비스 재시작 + Prisma client 재생성 필수

---

## 테스트용 쿼리

### INSERT 샘플

**`public.spx_audit_events`** — admin 카테고리 이벤트:
```sql
INSERT INTO public.spx_audit_events (
  id, occurred_at, category, action,
  actor_type, actor_id, actor_email,
  target_type, target_id, target_name,
  status, details, source, tenant_id
) VALUES (
  'test_' || gen_random_uuid()::text,
  NOW(),
  'admin',
  'app_create',
  'account',
  '559f9254-4c3d-489d-9815-1f05ad03a011',
  'test@test.com',
  'app',
  'app_uuid_xxx',
  'My Test App',
  'success',
  '{"mode": "chat", "description": "테스트 앱"}'::jsonb,
  'manual',
  'tenant_uuid_xxx'
);
```

**`public.spx_audit_events`** — security 글로벌 이벤트 (tenant_id NULL):
```sql
INSERT INTO public.spx_audit_events (
  id, occurred_at, category, action,
  actor_type, ip_address, status,
  target_type, target_name, details, source
) VALUES (
  'test_' || gen_random_uuid()::text,
  NOW(),
  'security',
  'auth_failed',
  'api',
  '192.168.1.100',
  'failed',
  'endpoint',
  'POST /v1/chat-messages',
  '{"method": "POST", "uri": "/v1/chat-messages", "httpStatus": 401}'::jsonb,
  'nginx_log'
);
```

**`public.spx_system_logs`** — 컨테이너 stdout 라인:
```sql
INSERT INTO public.spx_system_logs (
  id, occurred_at, source, log_level, message, raw, batch_date
) VALUES (
  'test_' || gen_random_uuid()::text,
  NOW(),
  'docker-api-1',
  'INFO',
  'Test log message from API',
  '2026-04-30T10:00:00.000Z 2026-04-30 10:00:00 INFO [Dummy] Test log message from API',
  CURRENT_DATE
);
```

### SELECT 샘플

**최근 이벤트 10건**:
```sql
SELECT occurred_at, category, action, actor_email, target_name, status
FROM public.spx_audit_events
ORDER BY occurred_at DESC
LIMIT 10;
```

**카테고리별 집계**:
```sql
SELECT category, action, count(*)
FROM public.spx_audit_events
GROUP BY category, action
ORDER BY category, count(*) DESC;
```

**source별 분포** (어떤 경로로 수집됐나):
```sql
SELECT source, count(*), max(occurred_at) AS latest
FROM public.spx_audit_events
GROUP BY source;
```

**특정 워크스페이스의 admin 이벤트** (tenant_id 격리 확인용):
```sql
SELECT occurred_at, action, actor_email, target_name
FROM public.spx_audit_events
WHERE category = 'admin'
  AND (tenant_id = 'tenant_uuid_xxx' OR tenant_id IS NULL)
ORDER BY occurred_at DESC
LIMIT 20;
```

**오늘 발생한 ERROR 레벨 시스템 로그**:
```sql
SELECT occurred_at, source, message
FROM public.spx_system_logs
WHERE log_level = 'ERROR'
  AND batch_date = CURRENT_DATE - INTERVAL '1 day'
ORDER BY occurred_at DESC;
```

**컨테이너별 로그 적재 현황**:
```sql
SELECT source, count(*) AS lines, min(occurred_at) AS first, max(occurred_at) AS last
FROM public.spx_system_logs
GROUP BY source
ORDER BY lines DESC;
```

**일일 배치 실행 결과**:
```sql
SELECT batch_date, source, status, line_count, finished_at - started_at AS duration
FROM public.spx_system_log_batch
ORDER BY batch_date DESC, source;
```

**collector 진행 상황**:
```sql
SELECT collector_name, last_run_at, cursor_value
FROM public.spx_collector_state
ORDER BY last_run_at DESC;
```

**Generated Column 채워졌는지 확인** (5/15 추가):
```sql
SELECT action, app_mode_d, triggered_from_d, target_app_id, total_tokens_d
FROM public.spx_audit_events
WHERE occurred_at > NOW() - INTERVAL '1 hour'
  AND action IN ('message_send','workflow_execute','workflow_node_execute','conversation_start')
ORDER BY occurred_at DESC
LIMIT 20;
```

**테스트 데이터 정리** (manual source만):
```sql
DELETE FROM public.spx_audit_events WHERE source = 'manual';
DELETE FROM public.spx_system_logs WHERE id LIKE 'test_%';
```

---

## 각 액션별 발생 시키는 법

### `source = 'dify_db'` (5분 폴링) — Dify 행위로 자동 발생

폴링이 5분마다 도니까 실제로 Dify에서 행동 후 약 5분 이내 spx_audit_events에 들어옴.

| 액션 | 어떻게 발생시키나 |
|---|---|
| `workflow_execute` | Dify Studio에서 워크플로우/챗플로우 앱의 "Run" 버튼 클릭 |
| `workflow_node_execute` | 워크플로우 실행 시 각 노드(LLM, IF, Code 등) 자동 발생 |
| `message_send` | Dify 챗 앱에서 메시지 전송 |
| `conversation_start` | 챗 앱에서 새 대화 시작 |
| `app_create` / `app_update` | Studio에서 앱 생성, 또는 기존 앱 설정 변경 후 저장 |
| `workflow_publish` / `workflow_draft` | 워크플로우 편집 후 "Publish" 또는 자동 draft 저장 |
| `member_join` / `member_change` | 워크스페이스 설정 → 멤버 추가/role 변경 |
| `prompt_update` | basic Chatbot/Text Generator 앱의 시스템 프롬프트 편집 후 저장 (chatflow/workflow는 발생 X) |
| `dataset_create` | 지식 메뉴 → "지식 생성" |
| `document_upload` | 데이터셋에 문서 업로드 |
| `api_token_create` | 앱 → "API 액세스" → "+ 새 비밀 키 생성" |
| `provider_model_add` / `provider_model_update` | 설정 → 모델 공급자 → 모델 추가/수정 |
| `message_feedback` | 챗 앱에서 답변에 👍 또는 👎 클릭 |

### `source = 'pg_trigger'` (실시간) — Dify에서 DELETE/role 변경 시

> trigger 함수: `public.spx_log_dify_change()`. trigger 5종은 위 § 6 참조.

| 액션 | 어떻게 |
|---|---|
| `document_delete` | 데이터셋에서 문서 삭제 |
| `dataset_delete` | 지식 메뉴에서 데이터셋 삭제 |
| `app_delete` | Studio에서 앱 삭제 |
| `api_token_delete` | API 액세스에서 토큰 폐기 |
| `member_remove` | 멤버 설정에서 멤버 제거 |
| `member_role_change` | 멤버의 role 변경 (예: editor → admin) |

또는 SQL 직접 실행:
```sql
-- 예: API 토큰 강제 삭제 → trigger 발동
DELETE FROM public.api_tokens WHERE id = '<token-uuid>';
```

### `source = 'nginx_log'` (실시간) — HTTP 요청 발생 시

| 액션 | 어떻게 |
|---|---|
| `auth_failed` | 인증 안 한 채 `/v1/*` 호출 → 401/403 |
| `api_call` | `/v1/chat-messages`, `/v1/completion-messages`, `/v1/workflows/run` 호출 |
| `rate_limit_exceeded` | nginx에서 429 응답 (기본 셋업엔 rate limit 없으므로 일시적 limit_req 추가 필요) |

```bash
# auth_failed 발생
curl -X POST http://localhost/v1/chat-messages

# api_call 발생 (auth 없으면 auth_failed로 분류 → status 401이라 우선순위 높음)
curl -H "Authorization: Bearer <invalid_token>" http://localhost/v1/chat-messages

# rate_limit_exceeded 시연 — 임시 limit 추가:
# 1. docker/nginx/conf.d/default.conf 의 location / 블록에 한 줄 추가:
#    limit_req zone=test burst=2 nodelay;
# 2. 새 conf 파일 docker/nginx/conf.d/zz-rate-limit.conf 생성:
#    limit_req_zone $binary_remote_addr zone=test:10m rate=2r/s;
#    limit_req_status 429;
# 3. docker exec docker-nginx-1 nginx -s reload
# 4. for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code} " http://localhost/; done
# 5. 끝나면 두 변경 원복 후 reload
```

### `source = 'dify_audit_app'` (실시간) — audit 페이지 사용 시

| 액션 | 어떻게 |
|---|---|
| `audit_login` | `/audit` 접속 → Keycloak 로그인 → owner/admin role이면 통과 |
| `audit_login_denied` | editor/normal role 사용자가 로그인 시도 (현재 워크스페이스 권한 부족) |
| `audit_logout` | `/audit` 헤더의 "로그아웃" 버튼 클릭 |
| `audit_list_view` | `/audit` 페이지 접속 또는 검색/필터 변경/페이지 이동 |
| `audit_detail_view` | 목록에서 행 클릭 → `/audit/[id]` 진입 |
| `audit_export` | 우상단 "내보내기" → CSV/JSON 다운로드 |

### `source = 'manual'` — 수동 SQL INSERT

위쪽 "테스트용 쿼리" 섹션의 INSERT 예제 사용.

### `spx_system_logs` — 일일 배치

| 발생 방법 | 설명 |
|---|---|
| 자동 cron | 매일 새벽 1시에 자동 실행 (어제 분 적재) |
| 수동 트리거 | API 호출로 특정 날짜 강제 실행 |

수동 트리거 (브라우저 콘솔에서, 로그인된 상태):
```js
fetch('/api/audit/system-logs/collect?date=2026-04-30', { method: 'POST' })
  .then(r => r.json()).then(console.log)
```

또는 curl with cookie:
```bash
curl -X POST "http://localhost:3000/api/audit/system-logs/collect?date=2026-04-30" \
  --cookie "authjs.session-token=<로그인 쿠키>"
```

### 검증 — 발생 후 확인

이벤트 적재 확인:
```sql
SELECT source, action, count(*), max(occurred_at) AS latest
FROM public.spx_audit_events
WHERE occurred_at > NOW() - INTERVAL '10 minutes'
GROUP BY source, action
ORDER BY latest DESC;
```

system_logs 적재 확인:
```sql
SELECT source, count(*), max(occurred_at) AS latest
FROM public.spx_system_logs
WHERE batch_date = CURRENT_DATE - INTERVAL '1 day'
GROUP BY source;
```
