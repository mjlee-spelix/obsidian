---
tags: [dify, 개발, AI-Agent]
date: 2026-04-22
---
# Dify - 토큰 데이터 저장 흐름 (동기·비동기)

## 핵심
- 토큰 수는 LLM Provider의 API 응답 `usage` 필드에서 가져오고, 없으면 tiktoken으로 직접 계산
- 채팅 메시지 토큰은 **동기적으로 즉시** DB에 저장되고, 워크플로우 토큰은 **Celery 비동기 태스크**로 저장됨
- 워크플로우 쪽은 worker가 처리하기 전까지 DB에 반영이 안 되어 있을 수 있음 (약간의 지연)

## 상세

### 토큰 수의 출처

```
LLM Provider 응답 (OpenAI, vLLM 등)
  → prompt_tokens, completion_tokens, total_price
  → LLMResult.usage 객체에 담김
```

- LLM 호출: `api/core/model_manager.py` → `invoke_llm()`
- 응답에 토큰 수가 없는 경우: `model_instance.get_llm_num_tokens()`로 tiktoken 기반 직접 계산 (폴백)
- Agent 모드에서는 여러 LLM 호출의 토큰을 누적 합산 (`increase_usage()`)

### 경로 1: 채팅 메시지 → 동기 저장

```
사용자 메시지 전송
  → model_manager.invoke_llm()
  → LLM 응답 수신 (usage 포함)
  → Message 객체에 토큰/비용 세팅
  → session.commit() 즉시 반영
```

**저장 코드 위치:**
- 일반 챗/완성: `api/core/app/task_pipeline/easy_ui_based_generate_task_pipeline.py` (391~405행)
- 고급 챗: `api/core/app/apps/advanced_chat/generate_task_pipeline.py` (957~967행)

**세팅되는 필드:**
```python
message.message_tokens = usage.prompt_tokens
message.answer_tokens = usage.completion_tokens
message.message_unit_price = usage.prompt_unit_price
message.answer_unit_price = usage.completion_unit_price
message.total_price = usage.total_price
message.currency = usage.currency
```

**타이밍**: LLM 응답 완료 직후, 요청 처리 안에서 동기적으로 커밋 → `messages` 테이블에 즉시 반영

### 경로 2: 워크플로우 → 비동기 저장 (Celery)

```
각 노드 실행 완료 / 워크플로우 전체 완료
  → persistence.py에서 토큰 집계
  → Celery 태스크 큐에 .delay()로 전달
  → worker 컨테이너가 백그라운드에서 DB 커밋
```

**집계 코드:** `api/core/app/workflow/layers/persistence.py`
- 각 노드의 `node_result.metadata`에 토큰 정보 포함
- `runtime_state.total_tokens`에 전체 노드 토큰 누적 합산
- 워크플로우 완료 시(성공/실패/중단/일시정지) `_populate_completion_statistics()`에서 최종 집계

**Celery 태스크:**

| 태스크 | 파일 | 대상 테이블 |
|--------|------|------------|
| `save_workflow_execution_task` | `api/tasks/workflow_execution_tasks.py` | `workflow_runs` |
| `save_workflow_node_execution_task` | `api/tasks/workflow_node_execution_tasks.py` | `workflow_node_executions` |

- 큐 이름: `workflow_storage`
- 실패 시 최대 3회 재시도 (60초 간격)
- 처리 컨테이너: worker

### Celery 태스크 큐 구조

Celery는 **비동기 작업 처리 프레임워크**로, 시간 걸리는 작업을 백그라운드에서 처리하기 위해 사용한다. 카페 주문에 비유하면:

```
손님(api)이 주문서를 카운터(Redis)에 놓으면
바리스타(worker)가 순서대로 꺼내서 처리한다
→ 손님은 커피 나올 때까지 기다리지 않고 자리에 감
```

**구성 요소 (Docker 컨테이너별 역할):**

| 역할 | 설명 | 컨테이너 |
|------|------|----------|
| Producer | 태스크를 만들어서 큐에 넣는 쪽 | `api` |
| Broker | 태스크 메시지를 보관하는 큐 | `redis` |
| Consumer | 큐에서 태스크를 꺼내 실행하는 쪽 | `worker` |
| Scheduler | 정해진 시간마다 태스크를 자동 생성 (cron 역할) → [[4. 지식노트/Dify - Celery Beat 정기 백그라운드 작업.md|상세 노트]] | `worker_beat` |

**실제 흐름 (워크플로우 노드 토큰 저장 예시):**

```
1. 워크플로우 노드 실행 완료
   → api 컨테이너의 persistence.py

2. save_workflow_node_execution_task.delay(데이터)
   → .delay()가 Redis에 "이 작업 해줘" 메시지를 넣음
   → api 컨테이너는 여기서 끝, 다음 노드 실행으로 넘어감

3. worker 컨테이너가 Redis를 계속 감시하다가
   → 새 태스크 발견 → 꺼내서 실행

4. worker가 DB에 session.commit()
   → workflow_node_executions 테이블에 토큰 데이터 저장 완료
```

**채팅 vs 워크플로우 — 왜 방식이 다른가:**
- 채팅: LLM 호출 1번 → 저장 1번이라 간단하니까 api가 직접 저장 (동기)
- 워크플로우: 노드 여러 개가 연속 실행 → 매번 직접 저장하면 API 응답이 느려지니까 분리 (비동기)

### 저장 시점 요약

| 대상 | 저장 방식 | 시점 | 테이블 |
|------|----------|------|--------|
| 채팅 메시지 토큰 | 동기 (요청 내) | LLM 응답 직후 즉시 | `messages` |
| 워크플로우 실행 토큰 | 비동기 (Celery) | 실행 완료 후 큐잉 | `workflow_runs` |
| 노드별 토큰 | 비동기 (Celery) | 각 노드 완료 후 큐잉 | `workflow_node_executions` |

> [!warning] 통계 집계 시 주의
> 워크플로우 토큰 데이터는 Celery worker가 태스크를 처리하기 전까지 DB에 반영되지 않는다. 실시간 대시보드를 만들 경우 이 지연을 감안해야 함.

## 관련 노트
- [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
