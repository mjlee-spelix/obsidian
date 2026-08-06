# Dify AppMode별 동작 규칙

> 에이전트 참조용. 모든 집계 쿼리에서 이 분기 규칙을 따라야 함.

## 동작하는 모드 5개

| Mode | 값 | 설명 |
|------|-----|------|
| COMPLETION | `"completion"` | 프롬프트 1회 입력 → 결과 반환 |
| CHAT | `"chat"` | 기본 채팅 |
| AGENT_CHAT | `"agent-chat"` | 도구 사용 가능한 에이전트 채팅 |
| ADVANCED_CHAT | `"advanced-chat"` | 워크플로우 캔버스 + 채팅 UI |
| WORKFLOW | `"workflow"` | 워크플로우 캔버스, 1회성 실행 |

CHANNEL, RAG_PIPELINE은 enum만 존재, 실제 동작 안 함.

## 토큰 저장 위치

| | messages | workflow_runs | 저장 방식 |
|---|:---:|:---:|---|
| COMPLETION / CHAT / AGENT_CHAT | ✅ 토큰+금액 | ❌ | 동기 |
| ADVANCED_CHAT | ✅ 토큰+금액 | ✅ 토큰만 | 동기(messages) + 비동기(workflow_runs) |
| WORKFLOW | ❌ | ✅ 토큰만 | 비동기(Celery) |

## 쿼리 분기 규칙 (모든 집계에 적용)

```
COMPLETION / CHAT / AGENT_CHAT → messages 테이블만
  필터: invoke_from != 'debugger'

ADVANCED_CHAT → messages 테이블만 (!!!)
  필터: invoke_from != 'debugger'
  주의: workflow_runs에도 같은 값 있지만 messages만 읽어서 이중카운트 방지 (H-DASH-01)

WORKFLOW → workflow_runs 테이블만
  필터: triggered_from = 'app-run'
  주의: messages에 저장 안 됨
```

## ADVANCED_CHAT 이중카운트 설명 (H-DASH-01)

ADVANCED_CHAT은 두 테이블에 토큰이 저장됨:
- `messages.message_tokens + answer_tokens` (파이프라인이 동기 저장)
- `workflow_runs.total_tokens` (레이어가 비동기 저장)

두 값은 **사실상 같음** (같은 `graph_runtime_state.llm_usage`에서 나옴).
따라서 한쪽만 읽어야 함. 기존 Dify 프론트엔드는 `messages` 기준 → 우리도 `messages`만.

## 모델 정보 접근성

| 테이블 | model_provider | model_id |
|--------|:---:|:---:|
| messages | ✅ 있음 | ✅ 있음 |
| workflow_runs | ❌ 없음 | ❌ 없음 |
| workflow_node_executions | ❌ (JSON 내부) | ❌ (JSON 내부) |

→ 모델별 토큰 차트: CHAT 계열은 `messages.model_provider`로 GROUP BY 가능.
→ WORKFLOW 모드는 모델별 분리 불가 → "미분류" 버킷 (H-DASH-02)

## 디버깅 필터

```sql
-- messages: 디버깅 제외
WHERE invoke_from != 'debugger'

-- workflow_runs: 디버깅 제외
WHERE triggered_from = 'app-run'
```

이 필터 없으면 Dify Studio 테스트 실행이 통계에 혼입됨 (H-DASH-03).

## 비동기 저장 주의 (Celery)

WORKFLOW / ADVANCED_CHAT의 `workflow_runs` 저장은 비동기:
- CeleryRepository → Redis 큐 → worker가 DB 커밋
- 지연: worker idle 시 ms~초, 부하 시 수십 초, 재시도 60초 간격 × 3회
- 방금 실행한 토큰이 대시보드에 바로 안 보일 수 있음 (H-DASH-06)
