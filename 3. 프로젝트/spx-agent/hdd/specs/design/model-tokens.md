---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: 모델별 토큰 사용량
harness: [H-DASH-01, H-DASH-02, H-DASH-03, H-DASH-09, H-DASH-11, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
date: 2026-04-29
last_updated: 2026-05-13
---
# 모델별 토큰 사용량 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/model-tokens.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/model-tokens.md|Tasks]]

## API 엔드포인트

```
GET /console/api/dashboard/model-tokens
Query Parameters:
  - start: string (YYYY-MM-DD HH:mm)
  - end: string (YYYY-MM-DD HH:mm)
Authorization: 로그인 사용자 전체 (관리자 전용 아님)
```

## Response 타입 (TypeScript — 프론트)

```typescript
interface ModelTokensResponse {
  models: {
    model_provider: string;
    model_id: string;
    display_name: string;    // "GPT-4o" 또는 "Llama 3 (로컬)"
    is_local: boolean;
    total_tokens: number;
  }[];
  unclassified_workflow_tokens: number;  // H-DASH-02: 미분류
  total_tokens: number;                   // 차트 하단 합계 표시용
  period_label: string;                   // "지난 7 일" 등 — 합계 텍스트에 함께 표시
}
```

## Response 스키마 (Pydantic — 백엔드)

```python
from pydantic import BaseModel

class ModelTokenItem(BaseModel):
    model_provider: str
    model_id: str
    display_name: str         # "GPT-4o" 또는 "Llama 3 (로컬)"
    is_local: bool
    total_tokens: int

class ModelTokensResponse(BaseModel):
    models: list[ModelTokenItem]              # total_tokens 내림차순
    unclassified_workflow_tokens: int         # H-DASH-02: WORKFLOW 모델 분리 불가분
    total_tokens: int                          # 차트 하단 합계
    period_label: str                          # "지난 7 일" 등
```

## 데이터 소스

`audit_events` 단일 SoT. action = `message_send` (CHAT 계열) + `workflow_execute` (WORKFLOW).

> 가용성 매트릭스: `references/audit-details-spec.md § 3`

## 쿼리 설계

**모델별 토큰 (`message_send` 기반 — CHAT 계열 + ADVANCED_CHAT):**
```sql
SELECT details->>'modelProvider' AS model_provider,
       details->>'modelId'       AS model_id,
       SUM((details->>'totalTokens')::int) AS total_tokens
FROM audit_events
WHERE action = 'message_send'
  AND occurred_at BETWEEN :start AND :end
  AND details->>'invokeFrom' != 'debugger'                  -- H-DASH-03
  AND details->>'modelProvider' IS NOT NULL
  AND details->>'modelId' IS NOT NULL
GROUP BY details->>'modelProvider', details->>'modelId'
ORDER BY total_tokens DESC;
```

> `message_send`는 CHAT/AGENT_CHAT/ADVANCED_CHAT/COMPLETION 모드에서 발생. WORKFLOW 모드는 `message_send` 없음 → AppMode 분기 불필요 (H-DASH-01 자연스럽게 충족).

**미분류 워크플로우 토큰 (`workflow_execute` 기반, H-DASH-02):**
```sql
SELECT SUM((details->>'totalTokens')::int) AS total_tokens
FROM audit_events
WHERE action = 'workflow_execute'
  AND occurred_at BETWEEN :start AND :end
  AND details->>'appMode' = 'workflow'                       -- H-DASH-01: WORKFLOW만 (ADVANCED_CHAT의 workflow_execute 제외)
  AND details->>'triggeredFrom' != 'debugging';              -- H-DASH-03 (P0 collector 보강 후 가용)
```

> **collector 보강 의존성**:
> - `details->>'appMode'`: 4종 collector(messages/workflow-runs/workflow-nodes/conversations) 모두 P0 보강 필요 (H-CAND-audit-appmode-missing). 보강 전에는 WORKFLOW 모드 식별 불가 → "미분류" 분리 동작 안 함.
> - `details->>'triggeredFrom'`: `workflow-runs.ts` collector P0 보강 필요 (H-CAND-audit-wf-debug-filter-missing). 보강 전에는 workflow_execute의 디버깅 실행이 합계에 혼입.

### 서비스 레이어

```python
# api/services/admin/dashboard_model_tokens_service.py (폴더명 admin/ 유지 — CLAUDE.md 검토 사항)

LOCAL_PROVIDERS = {'ollama', 'xinference', 'localai'}  # 설정으로 분리 가능 (H-DASH-09)

class DashboardModelTokensService:
    def get_model_tokens(self, start, end) -> dict:
        """audit_events에서 모델별 토큰 + 미분류 워크플로우 토큰 집계."""
        # 1. message_send GROUP BY details->>'modelProvider', details->>'modelId'
        # 2. workflow_execute SUM (details->>'appMode' = 'workflow')
        # 3. is_local = model_provider in LOCAL_PROVIDERS
        # 4. display_name = f"{model_id} (로컬)" if is_local else model_id
        # 5. total_tokens = sum(models[*].total_tokens) + unclassified_workflow_tokens
        pass
```

## 프론트엔드 컴포넌트

```
web/app/components/admin/
├── model-tokens-chart/
│   └── index.tsx               ← ECharts 수평 막대 차트 (폴더명 admin/ 유지)
web/service/
└── use-model-tokens.ts         ← TanStack Query 훅
```

> 페이지 마운트: `web/app/(commonLayout)/dashboard/page.tsx` (톱레벨 `/dashboard` 라우트).

**ECharts 설정 핵심:**
```typescript
// 수평 막대: xAxis=value, yAxis=category. 차트 영역 고정 높이.
const option: EChartsOption = {
  yAxis: {
    type: 'category',
    data: modelNames,           // ["GPT-4o", "Llama 3 (로컬)", "미분류 (Workflow)", ...]
    inverse: true,              // 큰 값이 위로
  },
  xAxis: { type: 'value' },
  series: [{
    type: 'bar',
    data: tokenCounts,
    label: {
      show: true,
      position: 'right',
      formatter: (p) => formatCount(p.value),  // "12.4M" 형식 (KPI 카드와 동일 유틸 재사용)
      color: 'var(--color-text-secondary)',
    },
    itemStyle: {
      color: (p) => isLocal[p.dataIndex]
        ? 'var(--color-util-colors-teal-teal-500)'      // 로컬 모델
        : 'var(--color-util-colors-blue-blue-500)',     // 기본
    },
  }],
};
```

**합계 표시 (차트 하단):**
```tsx
<div className="px-6 pb-4 text-text-tertiary text-xs">
  합계: {formatCount(data.total_tokens)} tokens / {data.period_label}
</div>
```

> 정확값은 hover tooltip에 추가 (KPI 카드 숫자 hover 패턴과 동일).

**레이아웃 정책:**
- 차트 컨테이너는 고정 높이 (예: `h-[420px]`). 모델 수가 늘어도 카드 크기 변동 금지.
- 모델 수가 영역을 초과하면 차트 영역 내부 스크롤 (전역 레이아웃 정책).

**클릭 인터랙션:**
- 차트 막대/축/하단 합계 — 모두 클릭 동작 없음 (차트 드로어 보류, H-DASH-20).
- ECharts `silent: true` 또는 `series.cursor: 'default'`로 hover 외 인터랙션 차단.

**TanStack Query 훅:**
```typescript
const useModelTokens = (params: { start: string; end: string }) => {
  return useQuery<ModelTokensResponse>({
    queryKey: ['dashboard', 'model-tokens', params],
    queryFn: () => get('/console/api/dashboard/model-tokens', { params }),
    staleTime: 5 * 60 * 1000,
  });
};
```

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit details 가용성 매트릭스]]
- [[3. 프로젝트/spx-agent/references/dify-app-modes.md|AppMode 규칙]]
