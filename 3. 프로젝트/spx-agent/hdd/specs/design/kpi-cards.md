---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: KPI 카드 4종
harness: [H-DASH-01, H-DASH-03, H-DASH-04, H-DASH-05, H-DASH-07]
harness_candidates: [H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
date: 2026-04-28
last_updated: 2026-05-22
---
# KPI 카드 4종 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/kpi-cards.md|Tasks]]

## API 엔드포인트

```
GET /console/api/dashboard/kpi
Query Parameters:
  - start: string (YYYY-MM-DD HH:mm)
  - end: string (YYYY-MM-DD HH:mm)
Authorization: 로그인 사용자 전체 (AppInitializer 가드 외 별도 권한 게이트 없음)
```

> KPI 4 drill-through 상세(Top 앱 차트·표)는 별도 엔드포인트로 분리:
> - `GET /console/api/dashboard/kpi/app-stats` — drill-through 진입 시에만 호출 (kpi-drill-through design 참조)

## Response 타입 (TypeScript — 프론트)

```typescript
interface KpiResponse {
  total_objects: {                  // KPI 1 — Volume
    count: number;
    app_count: number;
    kb_count: number;
    tool_count: number;
    diff_percent: number | null;    // null = N/A
    diff_label?: string;            // "+신규" 등
  };
  dept_adopted_apps: {              // KPI 2 — Breadth
    count: number;                  // COUNT(DISTINCT app_id) for user's dept members
    diff_percent: number | null;
    diff_label?: string;
  };
  api_calls: {                      // KPI 3 — Depth
    count: number;
    rps_avg: number;                // count / 기간_초
    diff_percent: number | null;
    diff_label?: string;
  };
  top_app_calls: {                  // KPI 4 — Application Analysis
    count: number;                  // Top 앱의 호출 수
    app_id: string | null;          // 앱 ID (drill-through 시 표·차트 기준)
    app_name: string | null;        // subtitle 표시용
    diff_percent: number | null;
    diff_label?: string;
  };
}
```

## Response 스키마 (Pydantic — 백엔드)

> 위 TypeScript 타입과 1:1 대응. `services/admin/dashboard_kpi_schemas.py`에 정의.
> Dify 컨벤션: `from pydantic import BaseModel`.

```python
from pydantic import BaseModel

class KpiDetail(BaseModel):
    """일반 KPI 기본 필드"""
    count: int
    diff_percent: float | None = None
    diff_label: str | None = None

class TotalObjects(KpiDetail):
    """KPI 1 — Volume"""
    app_count: int
    kb_count: int
    tool_count: int

class DeptAdoptedApps(KpiDetail):
    """KPI 2 — Breadth (COUNT(DISTINCT app_id))"""
    pass

class ApiCalls(KpiDetail):
    """KPI 3 — Depth"""
    rps_avg: float

class TopAppCalls(KpiDetail):
    """KPI 4 — Application Analysis (Top 앱)"""
    app_id: str | None = None
    app_name: str | None = None

class KpiResponse(BaseModel):
    total_objects: TotalObjects
    dept_adopted_apps: DeptAdoptedApps
    api_calls: ApiCalls
    top_app_calls: TopAppCalls
```

> **검증 자동화**: `KpiResponse(...)` 생성 시 필드 누락/타입 불일치 즉시 에러.
> **JSON 직렬화**: `response.model_dump()` → Flask가 JSON으로 변환.
> **테스트**: 각 모델별 valid/invalid 케이스 + edge case (Top 앱 없음 → `app_id=None`, count=0).

## 쿼리 설계

> **마트 입력 원칙**: KPI는 `audit_events` 단일 SoT + RBAC JOIN을 우선 시도. 단 KPI 1(총 오브젝트)은 state 본질이라 `spx_resource_ownership` 직접 조회. 마트 ETL 보강(P0) 전 OLTP fallback 가능.

### 공통 디버깅 필터 (H-DASH-03)

audit_events 기반 쿼리는 모두 다음 필터를 적용:

```sql
WHERE details->>'invokeFrom' != 'debugger'
  AND COALESCE(details->>'triggeredFrom', '') != 'debugging'
```

- `triggeredFrom`은 workflow 계열 이벤트에서만 의미가 있음. collector가 해당 필드를 미수집할 가능성(H-CAND-audit-wf-debug-filter-missing) — 마트 ETL 검증 단계에서 표본 카운트로 확인.

### AppMode 분기 (H-DASH-01)

ADVANCED_CHAT 앱은 messages 계열 이벤트만 카운트. audit_events.details에서 `details->>'appMode'`로 분기:

```sql
-- API 호출 / Top 앱: ADVANCED_CHAT은 messages 이벤트만
WHERE (
  details->>'appMode' != 'advanced-chat'
  OR action LIKE 'message.%'  -- messages 계열만 통과
)
```

- `appMode` 필드 자체가 audit_events에 미수집될 위험 — H-CAND-audit-appmode-missing. collector 보강(P0) 전에는 `apps` 테이블 JOIN으로 fallback:

```sql
LEFT JOIN apps a ON a.id = (e.details->>'targetAppId')::uuid
WHERE (a.mode != 'advanced-chat' OR e.action LIKE 'message.%')
```

### KPI 1 — 총 오브젝트

```sql
-- 현재 전체 수
SELECT object_type, COUNT(*) AS cnt
FROM spx_resource_ownership
GROUP BY object_type;

-- 미배정 앱 수 (H-DASH-04)
SELECT COUNT(*) FROM apps a
WHERE NOT EXISTS (
  SELECT 1 FROM spx_resource_ownership o
  WHERE o.object_id = a.id AND o.object_type = 'app'
);

-- 증감 — 기간 내 신규 등록
SELECT COUNT(*) FROM spx_resource_ownership
WHERE created_at BETWEEN :start AND :end;

SELECT COUNT(*) FROM spx_resource_ownership
WHERE created_at BETWEEN :prev_start AND :prev_end;
```

> "신규"의 정의(ownership reg vs app creation)는 PM 확정 필요. 위 쿼리는 ownership.created_at 기준.

### KPI 2 — 총 이용 앱 수

```sql
-- 사용자 소속 부서의 구성원들이 기간 내 호출한 고유 앱 수
WITH user_depts AS (
  SELECT department_id
  FROM spx_department_members
  WHERE account_id = :current_user_account_id
),
dept_member_ids AS (
  SELECT DISTINCT dm.account_id
  FROM spx_department_members dm
  JOIN user_depts ud ON dm.department_id = ud.department_id
)
SELECT COUNT(DISTINCT e.details->>'targetAppId') AS count
FROM audit_events e
WHERE e.occurred_at BETWEEN :start AND :end
  AND e.actor_id IN (SELECT account_id FROM dept_member_ids)
  AND e.details->>'invokeFrom' != 'debugger'
  AND COALESCE(e.details->>'triggeredFrom', '') != 'debugging'
  AND e.details->>'targetAppId' IS NOT NULL;
```

- 사용자가 어떤 부서에도 속하지 않으면 `dept_member_ids`가 비어 결과 0 → 서비스 레이어에서 `diff_label="N/A"` 처리
- 다중 부서 소속 시 `user_depts`가 여러 row → 자연스러운 합집합 계산

### KPI 3 — API 호출

```sql
-- 기간 내 호출 수 (audit_events 기반, AppMode 분기 + 디버그 필터)
SELECT COUNT(*) AS count
FROM audit_events e
LEFT JOIN apps a ON a.id = (e.details->>'targetAppId')::uuid
WHERE e.occurred_at BETWEEN :start AND :end
  AND e.action IN ('message.created', 'workflow_run.completed', /* ... */)
  AND (a.mode != 'advanced-chat' OR e.action LIKE 'message.%')  -- H-DASH-01
  AND e.details->>'invokeFrom' != 'debugger'                     -- H-DASH-03
  AND COALESCE(e.details->>'triggeredFrom', '') != 'debugging';
```

- RPS 평균: 서비스 레이어에서 `count / (end - start).total_seconds()` 단순 계산. 0초/음수 방어 필요
- nginx access log 기반 외부 API key 호출의 부서별 분류 가능성은 PM 확정 후 보강

### KPI 4 — Top 앱 호출

```sql
-- 기간 내 호출 수 최대 앱 선정
SELECT
  (e.details->>'targetAppId')::uuid AS app_id,
  COALESCE(a.name, e.details->>'targetAppName', '(이름 없음)') AS app_name,
  COUNT(*) AS count
FROM audit_events e
LEFT JOIN apps a ON a.id = (e.details->>'targetAppId')::uuid
WHERE e.occurred_at BETWEEN :start AND :end
  AND e.action IN ('message.created', 'workflow_run.completed', /* ... */)
  AND (a.mode != 'advanced-chat' OR e.action LIKE 'message.%')  -- H-DASH-01
  AND e.details->>'invokeFrom' != 'debugger'                     -- H-DASH-03
  AND COALESCE(e.details->>'triggeredFrom', '') != 'debugging'
  AND e.details->>'targetAppId' IS NOT NULL
GROUP BY app_id, app_name
ORDER BY count DESC
LIMIT 1;
```

- 기간 내 어떤 호출도 없으면 결과 0 row → 서비스 레이어에서 `count=0, app_id=None, app_name=None, diff_label="N/A"` 처리
- drill-through 진입 시 LIMIT 1 → LIMIT 10 + 표/차트 추가 데이터 — 별도 엔드포인트(`/kpi/app-stats`)에서 처리. 상세는 kpi-drill-through design 참조

### 증감률 계산 (서비스 레이어)

```python
def calc_diff(current: int, previous: int) -> dict:
    if previous == 0 and current > 0:
        return {"diff_percent": None, "diff_label": "+신규"}
    if previous == 0 and current == 0:
        return {"diff_percent": None, "diff_label": "-"}
    pct = round((current - previous) / previous * 100, 1)
    return {"diff_percent": pct}

def calc_diff_with_history_check(
    current: int,
    previous: int,
    prev_window_has_data: bool,
) -> dict:
    """이전 기간 데이터가 Celery Beat에 의해 삭제됐으면 N/A (H-DASH-05)"""
    if not prev_window_has_data:
        return {"diff_percent": None, "diff_label": "N/A"}
    return calc_diff(current, previous)
```

## 프론트엔드 컴포넌트

```
web/app/components/admin/
├── dashboard-controls/         ← 페이지 헤더 슬롯 컨트롤 (별도 spec)
└── kpi-section/
    ├── index.tsx               ← KPI 4개 카드 컨테이너 (grid, 고정 높이)
    ├── kpi-card.tsx            ← 개별 카드 (좌우 분리 레이아웃) + 내부에 `DiffBadge` inline 정의
    ├── format-number.ts        ← 숫자 포맷 유틸 (raw / K / M + 정확값)
    ├── use-kpi-drill-through.ts ← KpiMetric 토글 훅 (drill-through spec 참조)
    └── types.ts                ← KpiResponse, KpiCardProps, KpiMetric
```

> `DiffBadge`는 별도 파일이 아니라 `kpi-card.tsx` 내부 `function DiffBadge(...)` 로 정의됨. 분기 6종(▴/▾/`-`/`+신규`/`N/A`/임의 라벨)을 한 컴포넌트로 처리.

**KpiCardProps:**

```typescript
interface KpiCardProps {
  title: string;
  tooltip: string;              // (?) hover 시 표시
  periodLabel: string;          // "지난 7 일"
  count: number;                // KPI 1=new_count, KPI 2·3=count, KPI 4=topApp.calls
  subtitle?: string;            // KPI 4 앱명 — 큰 자리에 truncate + hover 풀텍스트로 렌더
  diffPercent: number | null;
  diffLabel?: string;           // "+신규" / "-" / "N/A" / KPI 4의 호출 수 문자열
  newCount?: number;            // diff_label="+신규" 시 동반 수치 (KPI 1 전용)
  isActive?: boolean;           // drill-through 선택 상태 (ring-inset ring-2 강조)
  isClickable?: boolean;        // drill-through 가능 여부
  metricKey?: KpiMetric;        // 'objects'|'users'|'calls'|'apps'
  onActivate?: (m: KpiMetric) => void;
}
```

> KPI 1의 App/KB/Tool 분해(`details` prop)는 제거됨 — 부서별 오브젝트 차트의 카드 헤더 React 범례로 이동. 따라서 `KpiDetailItem` 타입도 미사용.

**활성 상태 시각 강조 (active card):**

```tsx
cn(
  'flex flex-col gap-1 rounded-xl border border-solid border-components-card-border bg-components-card-bg px-6 py-4 shadow-sm transition-all duration-200 ease-in-out',
  isActive
    ? 'border-transparent ring-inset ring-2 ring-components-button-primary-bg-hover shadow-lg'
    : '',
  isClickable && !isActive && 'cursor-pointer hover:shadow-lg',
)
```

- 활성 시 배경 토큰 변경 없음 (`bg-components-card-bg-hover` 미적용). `ring-inset` + `shadow-lg`만으로 강조.

**좌우 분리 내부 구조:**

```tsx
<div className="flex items-start justify-between">
  {/* 좌 컬럼: 제목 + 툴팁 / 기간 라벨 */}
  <div className="flex flex-col gap-1">
    <div className="flex items-center gap-1">
      <span className="system-sm-semibold text-text-secondary">{title}</span>
      <Tooltip>{/* (?) */}</Tooltip>
    </div>
    <div className="system-2xs-regular text-text-tertiary">{periodLabel}</div>
  </div>
  {/* 우 컬럼: 큰 숫자 또는 앱명 / 증감 뱃지 */}
  <div className="flex flex-col items-end gap-0.5 min-w-0 max-w-[55%]">
    {subtitle
      ? <span className="block max-w-full truncate text-2xl font-semibold text-text-primary" title={subtitle}>{subtitle}</span>
      : <span className="text-2xl font-semibold text-text-primary">{formattedCount}</span>}
    <DiffBadge diffPercent={diffPercent} diffLabel={diffLabel} newCount={newCount} />
  </div>
</div>
```

**숫자 포맷 유틸:**

```typescript
// format-number.ts
export const formatCount = (n: number): string => {
  if (n < 10_000) return n.toLocaleString();
  if (n < 1_000_000) return `${(n / 1_000).toFixed(1).replace(/\.0$/, '')}K`;
  return `${(n / 1_000_000).toFixed(1).replace(/\.0$/, '')}M`;
};

export const formatExact = (n: number): string => n.toLocaleString();
```

**TanStack Query 훅:**

```typescript
// use-dashboard-kpi.ts
const useDashboardKpi = (params: { start: string; end: string }) => {
  return useQuery<KpiResponse>({
    queryKey: ['dashboard', 'kpi', params],
    queryFn: () => get('/dashboard/kpi', { params }),  // baseURL /console/api/
    staleTime: 5 * 60 * 1000,
  });
};
```

> 페이지 헤더의 새로고침 버튼은 `queryClient.invalidateQueries({ queryKey: ['dashboard'] })`로 KPI 포함 일괄 무효화.
> 기간 변경 시 URL query string이 갱신되고, `params` 객체 ref가 바뀌면서 자동 refetch.

## 고정 높이 / 레이아웃 토큰

- 4 카드 컨테이너: `grid grid-cols-4 gap-4`, 각 카드 `h-[140px]` (예시) — 디자인 토큰화 검토
- 카드 내부: `flex flex-col justify-between` 구조로 위쪽(제목+기간) / 가운데(큰 숫자+뱃지) / 아래쪽(부가정보 or subtitle) 고정 슬롯
- subtitle 폭 초과 시 `truncate` + `title={subtitle}`로 hover 풀텍스트

## 디자인 토큰 매핑

> 상세: [[4. 지식노트/Dify - 디자인 토큰과 차트·카드 색상 패턴.md]]

| 용도 | 토큰 | Tailwind |
|------|------|---------|
| 카드 배경 | `--color-components-chart-bg` | `bg-components-chart-bg` |
| 제목 | `--color-text-secondary` | `text-text-secondary` |
| 기간 라벨 / 부가 정보 숫자 / subtitle | `--color-text-tertiary` | `text-text-tertiary` |
| (?) 아이콘 | `--color-text-quaternary` | `text-text-quaternary` |
| 큰 숫자 | `--color-text-primary` | `text-text-primary` |
| 증감 상승 | `--color-text-success` | `text-text-success` |
| 증감 하락 | `--color-text-destructive` | `text-text-destructive` |
| Active 테두리 | `border-components-button-primary-bg-hover` | (drill-through 강조) |
| App | `util-colors-blue-blue-500` | `text-util-colors-blue-blue-500` |
| KB | `util-colors-teal-teal-500` | `text-util-colors-teal-teal-500` |
| Tool | `util-colors-orange-orange-500` | `text-util-colors-orange-orange-500` |

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]]
- [[3. 프로젝트/spx-agent/references/dify-db-schema.md|Dify DB 스키마]]
- [[3. 프로젝트/spx-agent/references/dify-app-modes.md|AppMode 규칙]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit_events details 매트릭스]]
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|RBAC 스키마]]
