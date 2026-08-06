---
tags: [dify, 개발, AI-Agent]
date: 2026-04-28
---
# Dify - workflow_node_executions에서 모델별 토큰 추출

## 핵심
- `workflow_runs`에는 모델 정보 없이 `total_tokens` 합산만 저장되어 WORKFLOW 모드에서 모델별 토큰 분리가 안 됨
- `workflow_node_executions`의 `process_data` JSON 필드에 `model_provider`, `model_name`이 저장되어 있어 **노드 단위로 모델 식별 가능**
- 토큰/비용은 같은 테이블의 `execution_metadata` JSON 필드에 `total_tokens`, `total_price`, `currency`로 저장
- JSON 파싱 + 집계라 쿼리가 무겁고 인덱스를 못 타므로, 캐시 또는 집계 테이블 고려 필요

## 상세

### 문제: WORKFLOW 모드 모델별 토큰 분리 불가

| 테이블 | 모델 정보 | 토큰 |
|--------|:---------:|:----:|
| `messages` | ✅ `model_provider`, `model_id` 컬럼 | ✅ |
| `workflow_runs` | ❌ 없음 | ✅ 합산만 |
| `workflow_node_executions` | ✅ `process_data` JSON 안에 있음 | ✅ `execution_metadata` JSON |

`messages` 테이블 기반인 COMPLETION / CHAT / AGENT_CHAT / ADVANCED_CHAT은 `model_provider` 컬럼으로 GROUP BY 하면 모델별 분리 가능. WORKFLOW 모드만 `workflow_runs`에 모델 정보가 없어서 문제.

### process_data에 저장되는 모델 정보

소스: `api/core/workflow/nodes/llm/node.py`

```python
process_data = {
    "model_mode": model_config.mode,
    "prompts": ...,
    "usage": jsonable_encoder(usage),
    "finish_reason": finish_reason,
    "model_provider": model_config.provider,  # "openai", "anthropic" 등
    "model_name": model_config.model,          # "gpt-4", "claude-3-sonnet" 등
}
```

### execution_metadata에 저장되는 토큰/비용

소스: `api/core/workflow/enums.py` — `WorkflowNodeExecutionMetadataKey` enum

```python
{
    "total_tokens": 1234,
    "total_price": 0.0025,
    "currency": "USD"
}
```

### 모델별 토큰 추출 쿼리 예시

```sql
SELECT 
    process_data::json->>'model_provider' AS model_provider,
    process_data::json->>'model_name'     AS model_name,
    SUM((execution_metadata::json->>'total_tokens')::bigint) AS total_tokens
FROM workflow_node_executions
WHERE node_type = 'llm'
  AND tenant_id = :tenant_id
  AND status = 'succeeded'
  AND created_at BETWEEN :start AND :end
GROUP BY model_provider, model_name;
```

### 주의사항

| 항목 | 내용 |
|------|------|
| JSON 파싱 비용 | `::json->>'key'` 사용, 전용 컬럼이 아니라 인덱스 못 탐 |
| 데이터 양 | 노드 수 × 실행 수만큼 행이 쌓여서 데이터 많으면 느림 |
| LLM 외 노드 | 질문분류기(`question-classifier`), 매개변수추출기(`parameter-extractor`) 등도 내부적으로 LLM 호출 — `node_type = 'llm'`만 필터하면 누락될 수 있음 |
| process_data 구조 | 노드 타입마다 다름 — LLM 노드는 `model_provider`/`model_name`이 있지만 다른 노드 타입은 키 이름이 다를 수 있어 추가 확인 필요 |

### 구현 방안 비교

| 방법 | 난이도 | 장단점 |
|------|:------:|--------|
| 워크플로우는 합산으로 표시 | 하 | 간단하지만 모델 구분 불가. "워크플로우 앱은 모델별 분리 불가" 안내 필요 |
| `workflow_node_executions` JSON 파싱 | 중 | **가능 확인됨.** 쿼리 무거움 → Redis 캐시(5분) 또는 집계 테이블 권장 |
| 저장 구조 수정 | 상 | `workflow_runs`에 모델별 토큰 필드 추가. Dify 코어 수정이라 유지보수 부담 |

현실적 추천: 기본은 합산으로 보여주되, 상세 드릴다운에서 `workflow_node_executions` 파싱으로 모델별 분리 제공. 캐시 필수.

## 관련 노트
- [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]
- [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]]
- [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
