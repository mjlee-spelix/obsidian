---
tags: [dify, 개발, AI-Agent]
date: 2026-04-27
---
# Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름

## 핵심
- 앱 모드에 따라 **토큰이 저장되는 테이블**과 **통계를 읽는 쿼리**가 다름
- 통계 시스템은 **메시지 기반**(8종)과 **워크플로우 기반**(4종)으로 완전 분리
- 워크플로우 내 LLM을 사용하는 모든 노드(LLM, 질문 분류기, 매개변수 추출기, 지식 검색 등)의 토큰이 동일하게 `llm_usage`와 `total_tokens` 양쪽에 집계됨 — 사실상 같은 값

## AppMode 종류

| Mode | 값 | 상태 | 설명 |
|------|-----|------|------|
| COMPLETION | `"completion"` | ✅ 동작 | 프롬프트 1회 입력 → 결과 반환 |
| CHAT | `"chat"` | ✅ 동작 | 기본 채팅 |
| AGENT_CHAT | `"agent-chat"` | ✅ 동작 | 도구(Tool) 사용 가능한 에이전트 채팅 |
| ADVANCED_CHAT | `"advanced-chat"` | ✅ 동작 | 워크플로우 캔버스로 만들지만 채팅 UI로 사용 |
| WORKFLOW | `"workflow"` | ✅ 동작 | 워크플로우 캔버스, 1회성 실행 |
| CHANNEL | `"channel"` | ❌ 미구현 | enum만 존재, 생성/처리 불가 |
| RAG_PIPELINE | `"rag-pipeline"` | ❌ 미구현 | enum만 존재, 생성/처리 불가 |

CHANNEL, RAG_PIPELINE은 `ALLOW_CREATE_APP_MODES`에서 제외, 백엔드에서 `ValueError` 발생, 프론트엔드 `AppModeEnum`에도 없음. **실제 동작하는 모드는 5개.**

## 파이프라인과 레이어

토큰 저장에 관여하는 구조가 두 가지 있음: **파이프라인**과 **레이어**.

### 파이프라인 (Pipeline) — 실행 끝난 후 결과 소비

실행이 완료된 뒤 큐에서 이벤트를 받아서 **사용자에게 응답 스트리밍 + messages 테이블에 저장**하는 역할.

| 파이프라인 | 대상 모드 | 하는 일 |
|-----------|----------|---------|
| `EasyUIBasedGenerateTaskPipeline` | COMPLETION, CHAT, AGENT_CHAT | LLM 1개 응답을 스트리밍하면서 `_save_message()`로 messages에 동기 저장 |
| `AdvancedChatAppGenerateTaskPipeline` | ADVANCED_CHAT | 워크플로우 엔진 결과를 스트리밍하면서 `_save_message()`로 messages에 동기 저장 |

WORKFLOW 모드는 채팅이 아니라 messages에 저장할 게 없으므로 **파이프라인이 없음**.

### 레이어 (Layer) — 실행 도중에 끼어드는 미들웨어

워크플로우 엔진(GraphEngine) **내부에서** 노드가 완료될 때마다 `on_event()`가 호출되어 **workflow_runs / workflow_node_executions에 저장**하는 역할.

| 레이어 | 대상 모드 | 하는 일 |
|--------|----------|---------|
| `WorkflowPersistenceLayer` | ADVANCED_CHAT, WORKFLOW | 노드 완료/워크플로우 완료 이벤트마다 실행 기록 저장 |

레이어 자체는 그냥 `repository.save(data)`를 호출할 뿐이고, **실제로 어떻게 저장되는지는 리포지토리가 결정**:

```
WorkflowPersistenceLayer → repository.save(data)
  ├→ CeleryRepository를 넣어주면    → Celery 큐에 넣고 끝 → worker가 나중에 DB 커밋 (비동기)
  └→ SQLAlchemyRepository를 넣어주면 → 바로 DB 커밋 (동기)
```

현재 Dify는 **CeleryRepository를 사용** → 워크플로우 토큰이 비동기로 저장됨.
비동기인 이유: 노드 10개를 거칠 때 매번 DB 커밋을 기다리면 실행이 느려지니까, Celery에 맡기고 바로 다음 노드로 넘어가는 것.
트레이드오프: DB 반영이 살짝 늦음 → 대시보드에서 방금 실행한 토큰이 바로 안 보일 수 있음.

### 모드별 동작 구조

**COMPLETION / CHAT / AGENT_CHAT** — 파이프라인만
```
사용자 요청 → LLM 호출 → 응답이 큐에 들어옴
  → EasyUIBasedGenerateTaskPipeline이 큐 소비
    → 스트리밍 응답 + _save_message()로 messages 동기 저장
```

**ADVANCED_CHAT** — 레이어 + 파이프라인 둘 다
```
사용자 요청 → 워크플로우 엔진 실행
  → [엔진 내부] 노드 완료될 때마다
    → WorkflowPersistenceLayer.on_event() → workflow_runs 비동기 저장
  → [엔진 완료 후] 결과가 큐에 들어옴
    → AdvancedChatAppGenerateTaskPipeline이 큐 소비
      → 스트리밍 응답 + _save_message()로 messages 동기 저장
```

**WORKFLOW** — 레이어만
```
사용자 실행 → 워크플로우 엔진 실행
  → [엔진 내부] 노드 완료될 때마다
    → WorkflowPersistenceLayer.on_event() → workflow_runs 비동기 저장
  (파이프라인 없음 — messages에 저장할 게 없으니까)
```

---

## 1단계: 토큰이 테이블에 저장되는 과정

### messages 테이블 (COMPLETION / CHAT / AGENT_CHAT / ADVANCED_CHAT)

대화가 발생하면 파이프라인이 `messages` 행에 저장:

| 필드 | 값의 출처 | 통계에서 쓰이는 곳 |
|------|----------|-------------------|
| `message_tokens` | LLM 응답 `usage.prompt_tokens` | 토큰 비용 집계 |
| `answer_tokens` | LLM 응답 `usage.completion_tokens` | 토큰 비용, TPS |
| `message_unit_price` | LLM 응답 | 금액 계산 |
| `answer_unit_price` | LLM 응답 | 금액 계산 |
| `total_price` | LLM이 계산한 금액 | 비용 집계 |
| `provider_response_latency` | `time.perf_counter() - start_at` | 평균 응답 시간, TPS |
| `conversation_id` | 메시지 생성 시 설정 | 일별 대화 수, 평균 메시지 수 |
| `from_end_user_id` | 메시지 생성 시 설정 | 일별 사용자 수 |

- **COMPLETION / CHAT / AGENT_CHAT**: `EasyUIBasedGenerateTaskPipeline`에서 LLM 1개의 usage를 직접 저장
- **ADVANCED_CHAT**: `AdvancedChatAppGenerateTaskPipeline`에서 `graph_runtime_state.llm_usage` (워크플로우 내 모든 LLM 사용 노드 합산)를 저장

### workflow_runs 테이블 (WORKFLOW / ADVANCED_CHAT)

워크플로우 실행이 완료되면 `WorkflowPersistenceLayer`가 저장:

| 필드 | 값의 출처 | 통계에서 쓰이는 곳 |
|------|----------|-------------------|
| `total_tokens` | `runtime_state.total_tokens` | 워크플로우 토큰 집계 |
| `created_by` | 실행한 사용자 ID | 일별 터미널, 평균 인터랙션 |
| `created_at` | 실행 시작 시점 | 일별 집계 기준 |
| `elapsed_time` | `finished_at - started_at` | (통계에선 안 쓰임) |

> [!info] llm_usage와 total_tokens는 같은 값
> graphon 엔진이 각 노드의 `NodeRunResult.llm_usage`를 자동 누적해서 `GraphRuntimeState.llm_usage`와 `GraphRuntimeState.total_tokens` **양쪽 다** 채움.
> LLM 노드뿐 아니라 **질문 분류기, 매개변수 추출기, 지식 검색** 등 LLM을 내부적으로 호출하는 노드도 모두 포함.
> 따라서 Advanced Chat에서 `messages`에 들어가는 토큰 수와 `workflow_runs`에 들어가는 토큰 수는 사실상 같음.

### 모드별 저장 요약

| | COMPLETION / CHAT / AGENT_CHAT | ADVANCED_CHAT | WORKFLOW |
|------|------|------|------|
| **messages** | ✅ 토큰 + 금액 | ✅ 토큰 + 금액 | ❌ 저장 안 됨 |
| **workflow_runs** | ❌ 저장 안 됨 | ✅ 토큰만 (금액 없음) | ✅ 토큰만 (금액 없음) |
| **파이프라인** | EasyUIBased | AdvancedChat | Workflow |
| **저장 방식** | 동기 | 동기(messages) + 비동기(workflow_runs) | 비동기(Celery) |

### 토큰 비용 계산 공식

**메시지 기반 (messages 테이블):**
```
token_count = message_tokens + answer_tokens
total_price = (message_tokens × message_unit_price × message_price_unit)
            + (answer_tokens × answer_unit_price × answer_price_unit)
```

**워크플로우 (workflow_runs 테이블):**
```
token_count = total_tokens  ← 모든 노드 합산
total_price = 없음 (금액 필드 자체가 없음)
```

## 2단계: 통계 쿼리가 데이터를 읽는 과정

### 메시지 기반 통계 8종 (`statistic.py` → `messages` 테이블)

공통 필터: `app_id = :app_id AND invoke_from != 'debugger'` + 기간 필터 + `GROUP BY date`

| 통계 | SQL 핵심 | 조인 테이블 | 대상 모드 |
|------|---------|-----------|----------|
| 일별 메시지 수 | `COUNT(*)` | - | 전체 |
| 일별 대화 수 | `COUNT(DISTINCT conversation_id)` | - | 전체 |
| 일별 사용자 수 | `COUNT(DISTINCT from_end_user_id)` | - | 전체 |
| 토큰 비용 | `SUM(message_tokens + answer_tokens)`, `SUM(total_price)` | - | 전체 |
| 대화당 평균 메시지 | `AVG(메시지 수 per 대화)` | `conversations` | CHAT, AGENT_CHAT, ADVANCED_CHAT |
| 평균 응답 시간 | `AVG(provider_response_latency) × 1000` (ms) | - | COMPLETION만 |
| TPS | `SUM(answer_tokens) / SUM(provider_response_latency)` | - | 전체 |
| 사용자 만족도 | `COUNT(좋아요) × 1000 / COUNT(메시지)` | `message_feedbacks` | 전체 |

> [!note] currency 하드코딩
> 토큰 비용 응답에서 `currency`는 DB 값을 무시하고 `"USD"`로 하드코딩됨.

### 워크플로우 통계 4종 (`workflow_statistic.py` → `workflow_runs` 테이블)

공통 필터: `tenant_id` + `app_id` + `triggered_from = 'app-run'` + 기간 필터 + `GROUP BY date`

| 통계 | SQL 핵심 | 대상 모드 |
|------|---------|----------|
| 일별 실행 수 | `COUNT(id)` | 전체 |
| 일별 터미널 수 | `COUNT(DISTINCT created_by)` | 전체 |
| 토큰 비용 | `SUM(total_tokens)` (금액 없음) | 전체 |
| 평균 인터랙션 | 서브쿼리: 사용자별 일별 실행 수 → `AVG` | WORKFLOW만 |

## 3단계: 모드별 전체 흐름 (저장 → 통계)

모드별 파이프라인/레이어 동작 구조는 위 "파이프라인과 레이어" 섹션 참고. 여기서는 **저장 → 통계 쿼리** 연결만 정리.

| 모드 | 저장 | 통계 |
|------|------|------|
| COMPLETION / CHAT / AGENT_CHAT | 파이프라인 → messages (동기) | `statistic.py` 8종 |
| ADVANCED_CHAT | 파이프라인 → messages (동기) + 레이어 → workflow_runs (비동기) | 프론트엔드는 `statistic.py`만 사용 |
| WORKFLOW | 레이어 → workflow_runs (비동기) | `workflow_statistic.py` 4종 |

## 내/외부 모델 토큰 추적

외부 모델(GPT 등 API)이든 내부 모델(젬마 등 셀프호스트)이든, Dify에서 LLM을 호출하면 응답의 `usage`가 동일한 흐름으로 DB에 저장됨. 모델 등록 방식과 무관하게 **토큰 추적은 이미 동작**.

### 모델 정보 저장 현황

| 테이블 | 모델 정보 | 토큰 |
|--------|:---:|:---:|
| `messages` | ✅ `model_provider`, `model_id` 필드 있음 | ✅ |
| `workflow_runs` | ❌ 없음 (합산만) | ✅ |
| `workflow_node_executions` | ❌ 없음 | ✅ (노드별) |

- `messages` 테이블에는 어떤 모델이 사용됐는지까지 기록됨 (예: `model_provider="openai"`, `model_id="gpt-4"`)
- 현재 통계 쿼리는 모델 구분 없이 전체 합산 (`GROUP BY date`만, 모델별 분리 없음)
- 필요하면 `messages.model_provider`로 `GROUP BY` 추가해서 모델별 통계 가능
- `workflow_runs`는 모델 정보가 없어서 워크플로우 모드에서는 모델별 분리 불가

## 대시보드 구현 시 주의점

- 부서별 토큰 합산 시 **앱 모드에 따라 조회 테이블 분기** 필요 (`messages` vs `workflow_runs`)
- WORKFLOW 모드는 **금액(total_price)이 없음** → 토큰 수만 보여주거나 별도 단가 적용
- ADVANCED_CHAT은 양쪽 테이블에 토큰이 있지만 **값은 같으므로** 한쪽만 읽으면 됨 (기존 프론트엔드는 `messages` 기준)
- 메시지 기반 통계는 `invoke_from != 'debugger'`로, 워크플로우 통계는 `triggered_from = 'app-run'`으로 디버깅 실행 제외

## 참고 코드
- `api/controllers/console/app/statistic.py` — 메시지 기반 통계 쿼리 8종
- `api/controllers/console/app/workflow_statistic.py` — 워크플로우 통계 쿼리 4종
- `api/repositories/sqlalchemy_api_workflow_run_repository.py` — 워크플로우 통계 실제 SQL
- `api/core/app/task_pipeline/easy_ui_based_generate_task_pipeline.py` — COMPLETION/CHAT/AGENT_CHAT 토큰 저장
- `api/core/app/apps/advanced_chat/generate_task_pipeline.py` — ADVANCED_CHAT 토큰 저장
- `api/core/app/workflow/layers/persistence.py` — 워크플로우 토큰 저장
- `api/core/app/workflow/layers/llm_quota.py` — 노드별 LLM quota 차감
- `api/core/repositories/celery_workflow_execution_repository.py` — Celery 비동기 저장 (현재 사용)
- `api/core/repositories/sqlalchemy_workflow_execution_repository.py` — DB 직접 저장 (동기)

## 관련 노트
- [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]
- [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[4. 지식노트/Dify - App·Dataset·Workflow ���브젝트 모델 구조.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 관련 소스 파일 목록.md]]
