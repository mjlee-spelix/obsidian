# Dify DB 스키마 — 대시보드 관련 테이블

> 에이전트 참조용. 대시보드 집계 쿼리에 필요한 테이블만 수록.

## 변경 이력

- **2026-05-18 (권대리님 `f88f4a6`)**: `accounts` → `spx_accounts` rename. 컬럼 동일, 테이블명만 변경. dify-audit collector raw query + RBAC FK 매핑(`department_members.account_id`) 모두 새 이름 사용. 마이그레이션 본 갱신 + dify-audit collector dist 재빌드 동반 (`docker compose build dify-audit + up -d --force-recreate`) — H-INFRA-03 대비

## 테이블 관계도

```
spx_accounts (id) ← Dify 기본 'accounts' rename (5/18 권대리님)
 └── spx_department_members (account_id) — RBAC 부서 매핑 (회사 표준, `spx_` 접두사)
 └── messages (from_end_user_id), workflow_runs (created_by) — 호출자 추적

apps (tenant_id)
 ├── conversations (app_id)
 │    └── messages (conversation_id, app_id)
 └── workflow_runs (app_id, workflow_id)
      └── workflow_node_executions (workflow_run_id)
```

## spx_accounts (Dify `accounts` 5/18 rename)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID PK | 계정 ID. RBAC `spx_department_members.account_id` + `spx_resource_ownership.owner_account_id` + `spx_resource_ownership.created_by` + dify-audit `actor_id`(actor_type='account' 시) 모두 이걸 참조 |
| email | String | 로그인 이메일 |
| name | String | 표시명 |
| password | String | 해시된 비밀번호 (Dify 표준) |
| ... | | (나머지 컬럼은 Dify 기본 동일, 변경 없음) |

**주의 — 5/18 rename 의 의미**:
- `accounts`는 Dify upstream 테이블인데 5/18 **`spx_` prefix 부여** (회사 fork에서 우리가 직접 손댄 표식 — audit 연동/Keycloak upsert 매핑 변경 등 fork 종속 변경 누적). **이후 2026-05-19 RBAC 5종(`spx_departments` 등)도 `spx_` 접두사로 통일** → 현재는 RBAC·accounts·마트 객체 전부 `spx_` 접두사. (~~"RBAC는 prefix 없음" 옛 서술 폐기~~ — `references/rbac-schema.md` 🔴 배너 참조)
- Dify upstream pull 시 `accounts` 테이블이 다시 등장하면 conflict — pull 시 권대리님 rename 패턴 따라 재적용 필요
- dify-audit collector raw query (`SELECT FROM accounts WHERE ...` 등) 모두 `spx_accounts`로 갱신됨. 옛 이름 잔존 시 `relation "accounts" does not exist` 폭발 → H-INFRA-03 (dist 빌드 캐시) 동반 발생 가능

## messages

채팅 메시지 단위. COMPLETION / CHAT / AGENT_CHAT / ADVANCED_CHAT에서 사용.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID PK | |
| app_id | UUID FK | 소속 앱 |
| conversation_id | UUID FK | 소속 대화 |
| model_provider | String | 모델 제공자 (예: "openai") |
| model_id | String | 모델 ID (예: "gpt-4") |
| message_tokens | Integer | 입력 토큰 수 |
| answer_tokens | Integer | 출력 토큰 수 |
| total_price | Numeric(10,7) | 총 비용 |
| provider_response_latency | Float | 응답 지연시간(초) |
| invoke_from | String | `'debugger'` / `'app-run'` 등 |
| from_end_user_id | UUID | 최종 사용자 ID (nullable) |
| from_source | String | 메시지 출처 |
| status | Enum | normal / error |
| created_at | DateTime | |

**주요 인덱스**: `(app_id, created_at)`, `(app_id, from_source, from_end_user_id)`

**토큰 합산**: `message_tokens + answer_tokens`

## workflow_runs

워크플로우 실행 단위. WORKFLOW / ADVANCED_CHAT에서 사용.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID PK | |
| tenant_id | UUID | 워크스페이스 |
| app_id | UUID FK | 소속 앱 |
| workflow_id | UUID | 워크플로우 정의 |
| triggered_from | Enum | `'debugging'` / `'app-run'` |
| created_by | UUID | 실행한 사용자 ID |
| total_tokens | BigInteger | 전체 노드 토큰 합산 |
| total_steps | Integer | 실행된 노드 수 |
| elapsed_time | Float | 소요 시간(초) |
| status | Enum | running / succeeded / failed / stopped |
| created_at | DateTime | |

**주요 인덱스**: `(tenant_id, app_id, triggered_from)`

**주의**: model_provider / model_id 필드 없음 → 모델별 분리 불가 (H-DASH-02)

## workflow_node_executions

노드 단위 실행. 모델별 토큰 드릴다운 시 필요 (선택적).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID PK | |
| workflow_run_id | UUID | 소속 워크플로우 실행 |
| node_type | String | llm / start / end / tool 등 |
| execution_metadata | JSON | `{total_tokens, total_price, currency}` |

**주의**: 모델 정보는 `process_data` JSON 내부에만 있음 → 파싱 필요, 성능 주의 (H-DASH-11)

## apps

앱 정보. mode 필드로 AppMode 분기.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID PK | |
| tenant_id | UUID | 워크스페이스 |
| name | String | 앱 이름 |
| mode | String | `'completion'` / `'chat'` / `'agent-chat'` / `'advanced-chat'` / `'workflow'` |
| created_at | DateTime | |

## 기존 집계 쿼리 패턴

```sql
-- 메시지 기반 토큰 집계 (기존 statistic.py)
SELECT DATE(created_at) AS date,
       SUM(message_tokens + answer_tokens) AS token_count
FROM messages
WHERE app_id = :app_id
  AND invoke_from != 'debugger'
GROUP BY date;

-- 워크플로우 실행 통계 (기존 workflow_statistic.py)
SELECT DATE(created_at) AS date,
       COUNT(*) AS run_count,
       SUM(total_tokens) AS total_tokens
FROM workflow_runs
WHERE app_id = :app_id
  AND triggered_from = 'app-run'
GROUP BY date;
```
