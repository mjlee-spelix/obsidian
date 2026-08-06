---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: 부서별 오브젝트 분포
harness: [H-DASH-04, H-DASH-13, H-DASH-20]
date: 2026-04-29
last_updated: 2026-05-22
---
# 부서별 오브젝트 분포 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dept-objects.md|Tasks]]
> 데이터 소스 검증: [[3. 프로젝트/spx-agent/references/objects-charts-feasibility.md|objects-charts-feasibility.md]]

## API 엔드포인트

```
GET /console/api/dashboard/dept-objects
Query Parameters: 없음 (현재 시점 기준 — 오브젝트는 state)
Authorization: 로그인 사용자 전체 (dataset_operator 포함)
```

> 보조 뷰가 추후 분리될 경우 같은 prefix 하위에 추가:
> - `GET /console/api/dashboard/dept-objects/top-owners?limit=N`
> - `GET /console/api/dashboard/dept-objects/new-creations?period_start=...&period_end=...`

## Response 타입 (TypeScript — 프론트)

```typescript
interface DeptObjectsResponse {
  departments: {
    department_id: string | null;       // 미배정 시 null
    department_name: string;             // 미배정 시 "미배정"
    app_count: number;
    kb_count: number;
    tool_count: number;
  }[];
}
```

> "미배정"은 별도 키 대신 `departments[]`의 한 행(`department_id: null, department_name: "미배정"`)으로 통합 표현. 차트가 다른 부서와 동일하게 렌더링하면 됨.

## Response 스키마 (Pydantic — 백엔드)

```python
from pydantic import BaseModel

class DeptObjects(BaseModel):
    department_id: str | None              # 미배정 시 None
    department_name: str                   # 미배정 시 "미배정"
    app_count: int
    kb_count: int
    tool_count: int

class DeptObjectsResponse(BaseModel):
    departments: list[DeptObjects]         # 합계 내림차순 정렬, 미배정은 마지막 행
```

## 쿼리 설계

### 기본 SQL 스켈레톤 (state 본질, audit 미사용)

```sql
SELECT
  ro.owner_department_id                       AS department_id,
  COALESCE(d.name, '미배정')                    AS department_name,
  ro.resource_type,                            -- 'app' | 'dataset' | 'tool'
  COUNT(*)                                     AS cnt
FROM spx_resource_ownership ro
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE ro.tenant_id = :tenant_id
GROUP BY ro.owner_department_id, d.name, ro.resource_type
ORDER BY department_name, ro.resource_type;
```

- `LEFT JOIN spx_departments` → `owner_department_id IS NULL`인 행은 `d.name`이 NULL → COALESCE로 "미배정"
- 결과를 서비스 레이어에서 부서별로 피봇하여 `{department_id, department_name, app_count, kb_count, tool_count}` 형태로 변환

### 운영 방어 버전 — `apps LEFT JOIN spx_resource_ownership` 패턴 (H-DASH-04 강화)

`spx_resource_ownership`에 행이 아예 없는 레거시 오브젝트도 "미배정"으로 포함하려면 OLTP 테이블 기준으로 LEFT JOIN. resource_type별로 UNION:

```sql
-- 앱
SELECT
  ro.owner_department_id                       AS department_id,
  COALESCE(d.name, '미배정')                    AS department_name,
  'app'                                        AS resource_type,
  COUNT(*)                                     AS cnt
FROM apps a
LEFT JOIN spx_resource_ownership ro
  ON ro.resource_id = a.id
 AND ro.resource_type = 'app'
 AND ro.tenant_id = a.tenant_id
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE a.tenant_id = :tenant_id
GROUP BY ro.owner_department_id, d.name

UNION ALL

-- 데이터셋
SELECT
  ro.owner_department_id,
  COALESCE(d.name, '미배정'),
  'dataset',
  COUNT(*)
FROM datasets ds
LEFT JOIN spx_resource_ownership ro
  ON ro.resource_id = ds.id
 AND ro.resource_type = 'dataset'
 AND ro.tenant_id = ds.tenant_id
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE ds.tenant_id = :tenant_id
GROUP BY ro.owner_department_id, d.name

UNION ALL

-- 도구 (tool_providers 또는 운영 정의 테이블)
SELECT
  ro.owner_department_id,
  COALESCE(d.name, '미배정'),
  'tool',
  COUNT(*)
FROM tool_providers tp
LEFT JOIN spx_resource_ownership ro
  ON ro.resource_id = tp.id
 AND ro.resource_type = 'tool'
 AND ro.tenant_id = tp.tenant_id
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE tp.tenant_id = :tenant_id
GROUP BY ro.owner_department_id, d.name;
```

> 본 패턴은 KPI 카드 "총 오브젝트"와의 합계 일치를 보장하기 위해 운영 환경에서 권장.

### 보조 뷰 SQL 윤곽 (선택)

**top-owners (Top N 소유자) — `owner_account_id` 사용**:
```sql
SELECT
  ro.owner_account_id                          AS owner_account_id,
  COALESCE(a.name, '(알 수 없음)')              AS owner_name,
  COALESCE(d.name, '미배정')                    AS department_name,
  COUNT(*)                                     AS object_count
FROM spx_resource_ownership ro
LEFT JOIN spx_accounts a ON a.id = ro.owner_account_id
LEFT JOIN spx_department_members dm
  ON dm.account_id = ro.owner_account_id AND dm.is_active = true
LEFT JOIN spx_departments d ON d.id = dm.department_id
WHERE ro.tenant_id = :tenant_id
  AND ro.owner_account_id IS NOT NULL          -- 부서 소유 리소스는 dept-cumulative에서 커버
GROUP BY ro.owner_account_id, a.name, d.name
ORDER BY object_count DESC
LIMIT :top_n;
```

> ⚠️ `created_by`(매핑률 29%) 사용 금지. `owner_account_id`(매핑률 94.7%, 의미상 소유자).

**dept-new-creations (기간별 신규 생성)**:
```sql
SELECT
  COALESCE(d.name, '미배정')                    AS department_name,
  ro.resource_type,
  COUNT(*)                                     AS new_count
FROM spx_resource_ownership ro
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE ro.tenant_id = :tenant_id
  AND ro.created_at >= :period_start
  AND ro.created_at <  :period_end
GROUP BY d.name, ro.resource_type
ORDER BY department_name, ro.resource_type;
```

> **PM 확인 후보**: `spx_resource_ownership.created_at`은 **소유권 등록 시점**이며 `apps.created_at`(앱 생성 시점)과 다를 수 있음. "신규"의 정의가 앱 생성 기준이면 `apps`(+ datasets, tool_providers) JOIN 필요 → Dify OLTP 의존 추가.

## 서비스 레이어

```python
class DashboardDeptObjectsService:
    def get_dept_objects(self, tenant_id: str) -> DeptObjectsResponse:
        """부서별 오브젝트 수 (spx_resource_ownership 단독 또는 OLTP LEFT JOIN 패턴)."""
        # 1. SQLAlchemy 모델: ResourceOwnership, Department (H-DASH-13 — 컬럼명 직접 사용 X)
        # 2. 운영 권장 경로는 apps/datasets/tool_providers LEFT JOIN spx_resource_ownership UNION ALL (H-DASH-04 강화)
        # 3. (department_id, resource_type) 행을 부서별로 피봇하여 {app_count, kb_count, tool_count} 생성
        # 4. 미배정 행(department_id IS NULL)은 마지막으로 정렬
        pass
```

## 프론트엔드 컴포넌트

```
web/app/components/admin/dept-objects/
├── index.tsx                       ← 카드 컨테이너 (고정 높이 + 내부 스크롤)
├── dept-objects-chart.tsx          ← ECharts 스택 막대 차트
└── (선택) hooks 분리는 services/use-admin-dashboard.ts 와 통일
```

```typescript
interface DeptObjectsChartProps {
  data: DeptObjectsResponse;
  // 클릭 핸들러 없음 — 차트 드로어 보류 (H-DASH-20)
}
```

### ECharts 설정 핵심 (수평 스택 막대)

```typescript
const option: EChartsOption = {
  // legend 옵션 미사용 — 카드 헤더 React 범례로 대체
  yAxis: {
    type: 'category',
    data: departmentNames,           // ["IT 본부", "마케팅", ..., "미배정"]
    inverse: true,                   // 큰 부서가 위로
  },
  xAxis: { type: 'value' },
  series: [
    {
      name: 'app',
      type: 'bar',
      stack: 'total',
      data: filter && filter !== 'app' ? appCounts.map(() => 0) : appCounts,
      itemStyle: { color: 'var(--color-util-colors-blue-blue-500)' },
      label: { show: false },
    },
    {
      name: 'kb',
      type: 'bar',
      stack: 'total',
      data: filter && filter !== 'kb' ? kbCounts.map(() => 0) : kbCounts,
      itemStyle: { color: 'var(--color-util-colors-teal-teal-500)' },
      label: { show: false },
    },
    {
      name: 'tool',
      type: 'bar',
      stack: 'total',
      data: filter && filter !== 'tool' ? toolCounts.map(() => 0) : toolCounts,
      itemStyle: { color: 'var(--color-util-colors-orange-orange-500)' },
      label: {
        show: true,                  // 마지막 series에만 합계 라벨
        position: 'right',
        formatter: (p) => totalByDept[p.dataIndex].toString(),
        color: 'var(--color-text-secondary)',
      },
    },
  ],
};
```

- `legend` 키는 옵션에서 제거. 범례는 카드 헤더의 React 컴포넌트로 처리(클릭 필터링과 i18n 라벨 통제).
- 차트 드로어 보류로 `onEvents` 클릭 핸들러는 부착하지 않음.

### 카드 헤더 React 범례 (5/22 신설)

```tsx
const types: Array<{ key: 'app' | 'kb' | 'tool'; color: string; label: string; count: number }> = [
  { key: 'app',  color: 'var(--color-util-colors-blue-blue-500)',   label: t('common.objectType.app'),  count: totalApp },
  { key: 'kb',   color: 'var(--color-util-colors-teal-teal-500)',   label: t('common.objectType.kb'),   count: totalKb },
  { key: 'tool', color: 'var(--color-util-colors-orange-orange-500)', label: t('common.objectType.tool'), count: totalTool },
];

const [filter, setFilter] = useState<'app' | 'kb' | 'tool' | null>(null);

return (
  <div className="flex items-center gap-3">
    {types.map(({ key, color, label, count }) => (
      <button
        key={key}
        type="button"
        className={cn(
          'flex items-center gap-1 cursor-pointer transition-opacity',
          filter && filter !== key ? 'opacity-40' : '',
        )}
        onClick={() => setFilter(prev => prev === key ? null : key)}
      >
        <span className="inline-block h-2 w-2 rounded-full" style={{ backgroundColor: color }} />
        <span className="text-text-tertiary">{label} {formatCount(count)}</span>
      </button>
    ))}
  </div>
);
```

- 상태는 컴포넌트 내부 `useState` 단독 — URL state 미사용
- `filter`가 변하면 `buildChartOption(data, filter)` 재계산 → 비선택 series는 zero-out

### 레이아웃 (고정 높이 + 내부 스크롤)

```tsx
<div className="h-[420px] flex flex-col">       {/* 카드 자체는 고정 높이 */}
  <header className="shrink-0">…</header>
  <div className="grow min-h-0 overflow-y-auto">  {/* 부서 행 많으면 영역 내부에서 스크롤 */}
    <DeptObjectsChart data={…} />
  </div>
</div>
```

### TanStack Query 훅

```typescript
const useDeptObjects = () => {
  return useQuery<DeptObjectsResponse>({
    queryKey: ['dashboard', 'dept-objects'],
    queryFn: () => get('/console/api/dashboard/dept-objects'),
    staleTime: 5 * 60 * 1000,
  });
};
```

> queryKey의 첫 세그먼트는 항상 `'dashboard'` — 페이지 헤더의 새로고침 버튼이 `queryKey: ['dashboard']` prefix로 일괄 invalidate.

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]] (마트 입력 규칙의 예외: 오브젝트 차트는 RBAC 직접 조회)
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|RBAC 스키마]] (회사 표준 `spx_` 접두사)
- [[3. 프로젝트/spx-agent/references/objects-charts-feasibility.md|오브젝트 차트 RBAC 단독 구현 가능성]]
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]] — H-DASH-04, H-DASH-13, H-DASH-20
