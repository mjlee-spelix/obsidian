---
tags: [dify, 개발, CS]
date: 2026-04-22
---
# Dify - 통계·토큰 DB 스키마 구조

## 핵심
- Dify는 채팅 메시지·워크플로우 실행·노드 실행 세 단계에서 토큰 수와 비용을 DB에 기록
- 모든 통계 데이터는 PostgreSQL에 저장되며, 기존 집계 쿼리는 `app_id` 기준으로만 동작
- 비용 계산은 입력/출력 토큰을 분리해서 각각 단가를 적용하는 구조

## 상세

### 테이블 관계도

```
Tenant (워크스페이스)
 └── apps (tenant_id)
      │
      ├── conversations (app_id)
      │    └── messages (conversation_id, app_id)
      │         └── message_agent_thoughts (message_id)
      │
      └── workflow_runs (app_id, workflow_id)
           ├── workflow_node_executions (workflow_run_id)
           ├── workflow_trigger_logs (workflow_run_id)
           └── workflow_archive_logs (workflow_run_id)
```

### 핵심 테이블

#### 1. `messages` — 채팅 메시지 단위 토큰

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `app_id` | StringUUID (FK) | 소속 앱 |
| `conversation_id` | StringUUID (FK) | 소속 대화 |
| `workflow_run_id` | StringUUID | 워크플로우 실행 연결 (nullable) |
| `message_tokens` | Integer | 입력(프롬프트) 토큰 수 |
| `answer_tokens` | Integer | 출력(응답) 토큰 수 |
| `message_unit_price` | Numeric(10,4) | 입력 토큰당 단가 |
| `message_price_unit` | Numeric(10,7) | 입력 가격 단위 (기본 0.001) |
| `answer_unit_price` | Numeric(10,4) | 출력 토큰당 단�� |
| `answer_price_unit` | Numeric(10,7) | 출력 가격 단위 (기본 0.001) |
| `total_price` | Numeric(10,7) | 총 비용 |
| `currency` | String(255) | 통화 (USD) |
| `provider_response_latency` | Float | 응답 지연시간(초) |
| `status` | Enum | normal / error |
| `created_at` | DateTime | |

**비용 계산식:**
```
total_price = (message_tokens × message_unit_price × message_price_unit)
            + (answer_tokens × answer_unit_price × answer_price_unit)
```

**주요 인덱스:**
- `(app_id, created_at)` — 앱별 기간 조회
- `(conversation_id)` — 대화별 조회
- `(app_id, from_source, from_end_user_id)` — 사용자별 조회

#### 2. `workflow_runs` — 워크플로우 실행 단위 토큰

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `tenant_id` | StringUUID | 워크스페이스 |
| `app_id` | StringUUID | 소속 앱 |
| `workflow_id` | StringUUID | 워크플로우 정의 |
| `triggered_from` | Enum | `debugging` / `app-run` |
| `status` | Enum | running / succeeded / failed / stopped |
| `total_tokens` | BigInteger | 전체 노드 토큰 합산 |
| `total_steps` | Integer | 실행된 노드 수 |
| `elapsed_time` | Float | 소요 시간(초) |
| `exceptions_count` | Integer | 에러 수 |
| `created_at` | DateTime | |
| `finished_at` | DateTime | |

**주요 인덱스:**
- `(tenant_id, app_id, triggered_from)` — 앱별 실행 유형 조회

#### 3. `workflow_node_executions` — 노드 단위 토큰

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `tenant_id` | StringUUID | 워크스페이스 |
| `app_id` | StringUUID | 소속 앱 |
| `workflow_run_id` | StringUUID | 소속 워크플로우 실행 |
| `node_id` | String(255) | 노드 식별자 |
| `node_type` | String(255) | llm, start, end, tool 등 |
| `index` | Integer | 실행 순서 |
| `status` | Enum | running / succeeded / failed |
| `elapsed_time` | Float | 노드 소요 시간(초) |
| `execution_metadata` | LongText (JSON) | `{total_tokens, total_price, currency}` |
| `created_at` | DateTime | |
| `finished_at` | DateTime | |

**주요 인덱스:**
- `(workflow_run_id)` — 실행별 노드 조회
- `(tenant_id, workflow_id, node_id, created_at DESC)` — 노드별 이력

### 보조 테이블

#### `conversations` — 대화 세션

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `app_id` | StringUUID | 소속 앱 |
| `mode` | Enum(AppMode) | workflow / chat / completion / agent-chat |
| `dialogue_count` | Integer | 대화 내 메시지 수 |
| `system_instruction_tokens` | Integer | 시스템 프롬프트 토큰 수 |

#### `message_agent_thoughts` — Agent 모드 LLM 호출 상세

| 컬럼              | 타입              | 설명     |
| --------------- | --------------- | ------ |
| `id`            | StringUUID (PK) |        |
| `message_id`    | StringUUID (FK) | 소속 메시지 |
| `message_token` | Integer         | 입력 토큰  |
| `answer_token`  | Integer         | 출력 토큰  |
| `total_price`   | Numeric         | 비용     |
| `latency`       | Float           | 응답 시간  |
| `tool`          | LongText        | 사용된 도구 |

#### `workflow_trigger_logs` — 트리거 실행 로그

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `tenant_id` | StringUUID | 워크스페이스 |
| `app_id` | StringUUID | 소속 앱 |
| `workflow_run_id` | StringUUID | 워크플로우 실행 연결 |
| `total_tokens` | Integer | 토큰 합계 |
| `elapsed_time` | Float | 소요 시간 |
| `status` | Enum | 실행 상태 |

#### `workflow_archive_logs` — 워크플로우 아카이브

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `workflow_run_id` | StringUUID | 원본 실행 |
| `run_total_tokens` | BigInteger | 아카이브된 토큰 수 |
| `run_total_steps` | Integer | 실행 단계 수 |
| `run_elapsed_time` | Float | 소요 시간 |
| `archived_at` | DateTime | 아카이브 시점 |

### 기존 집계 쿼리 패턴

앱별 일일 토큰/비용 집계 (statistic.py에서 사용):
```sql
SELECT DATE(created_at) AS date,
       SUM(message_tokens + answer_tokens) AS token_count,
       SUM(total_price) AS total_price
FROM messages
WHERE app_id = :app_id
  AND invoke_from != 'debugger'
GROUP BY date
ORDER BY date;
```

워크플로우 실행 통계:
```sql
SELECT DATE(created_at) AS date,
       COUNT(*) AS run_count,
       SUM(total_tokens) AS total_tokens,
       AVG(elapsed_time) AS avg_elapsed_time
FROM workflow_runs
WHERE app_id = :app_id
  AND triggered_from = 'app-run'
GROUP BY date;
```

## 관련 노트
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[4. 지식노트/PostgreSQL - search_path와 schema 네임스페이스.md]]
