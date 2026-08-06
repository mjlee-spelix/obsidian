---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: 부서별 활동 테이블
harness: [H-DASH-01, H-DASH-03, H-DASH-04, H-DASH-10, H-DASH-13, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
date: 2026-04-29
last_updated: 2026-05-13
---
# 부서별 활동 테이블 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dept-activity.md|Tasks]]

## API 엔드포인트

```
GET /console/api/dashboard/dept-activity
Query Parameters:
  - start: string (YYYY-MM-DD HH:mm)
  - end: string (YYYY-MM-DD HH:mm)
  - sort_by: string (기본 "api_calls", 가능: "api_calls", "token_usage")
  - sort_order: string (기본 "desc")
Authorization: 로그인 사용자 전체 (라우트 가드는 (commonLayout) AppInitializer)
```

> 페이지네이션 파라미터(`limit`/`offset`)는 Phase 1에서 미적용 — 부서 전체 반환. 부서 수가 운영상 증가하면 도입 검토.

## Response 타입 (TypeScript — 프론트)

```typescript
interface DeptActivityRow {
  department_id: string | null;   // null = 미배정 행
  department_name: string;        // "미배정" 포함
  new_apps: number;               // 기간 내 신규 App (resource_ownership)
  new_kbs: number;                // 기간 내 신규 KB
  new_tools: number;              // 기간 내 신규 Tool
  api_calls: number;
  token_usage: number;
}

interface DeptActivityResponse {
  rows: DeptActivityRow[];        // 부서 + "미배정" 행 (미배정은 항상 마지막)
  period: { start: string; end: string };
}
```

## Response 스키마 (Pydantic — 백엔드)

```python
from pydantic import BaseModel
from datetime import datetime

class DeptActivityRow(BaseModel):
    department_id: str | None      # None = 미배정
    department_name: str
    new_apps: int                  # 기간 내 신규 App (resource_ownership)
    new_kbs: int                   # 기간 내 신규 KB
    new_tools: int                 # 기간 내 신규 Tool
    api_calls: int
    token_usage: int

class PeriodWindow(BaseModel):
    start: datetime
    end: datetime

class DeptActivityResponse(BaseModel):
    rows: list[DeptActivityRow]
    period: PeriodWindow
```

## 데이터 소스

- **신규 App/KB/Tool**: `spx_resource_ownership` 직접 조회 (state 본질 — audit 아님). `created_at BETWEEN :start AND :end` + `resource_type` 분기
- **호출/토큰**: `audit_events` 단일 SoT + RBAC `spx_resource_ownership` LEFT JOIN (2026-05-12 이사님 결정)
- 부서 기준: **앱 소유 부서** (`spx_resource_ownership.owner_department_id`, `resource_type='app'`)
- 이벤트: `event_type IN ('message_send', 'workflow_execute')`
- 디버깅 필터 (H-DASH-03):
  - `message_send`: `details->>'invokeFrom' != 'debugger'`
  - `workflow_execute`: `details->>'triggeredFrom' != 'debugging'` — collector 보강 후 적용(H-CAND-audit-wf-debug-filter-missing)
- AppMode 이중카운트 방어 (H-DASH-01): collector `appMode` 적재(P0 보강) 완료 후 audit ETL에서 ADVANCED_CHAT은 `message_send`만, WORKFLOW는 `workflow_execute`만 카운트. 보강 전(H-CAND-audit-appmode-missing) 임시 우회: `apps` 테이블 JOIN으로 `app.mode` 분기

> `target_id`(top-level)에 `app_id`가 들어옴. 단 `workflow_node_execute`는 targetType='workflow_node'라 `details->>'appId'`로 추출 — 본 표는 사용하지 않음.

## 쿼리 설계

> CTE로 단계 분리 (H-DASH-10). audit 단일 SoT 채택으로 4단계 다단 JOIN → 2단계 집계로 축소.

**Step 1: 앱 → 부서 매핑 (CTE)**
```sql
WITH app_dept AS (
  SELECT
    ro.resource_id  AS app_id,
    ro.owner_department_id AS department_id
  FROM spx_resource_ownership ro
  WHERE ro.resource_type = 'app'
)
```

**Step 2: 부서별 신규 오브젝트 (resource_ownership 직접 — state 본질)**
```sql
, dept_new_objects AS (
  SELECT
    ro.owner_department_id AS department_id,
    COUNT(*) FILTER (WHERE ro.resource_type = 'app')     AS new_apps,
    COUNT(*) FILTER (WHERE ro.resource_type = 'dataset')  AS new_kbs,
    COUNT(*) FILTER (WHERE ro.resource_type = 'tool')    AS new_tools
  FROM spx_resource_ownership ro
  WHERE ro.created_at BETWEEN :start AND :end
  GROUP BY ro.owner_department_id
)
```

> ⚠️ PM 확인 — "신규" 시점: 현재 `resource_ownership.created_at`(소유권 등록) 기준. Dify 앱 생성 시점(`apps.created_at`)인지 확정 필요.

**Step 3: 부서별 활동 집계 (audit_events)**
```sql
, dept_activity AS (
  SELECT
    ad.department_id,                                -- NULL = 미배정 (LEFT JOIN fallback)
    COUNT(*)                                  AS api_calls,
    COALESCE(SUM((ae.details->>'totalTokens')::bigint), 0) AS token_usage
  FROM audit_events ae
  LEFT JOIN app_dept ad ON ad.app_id = ae.target_id      -- H-DASH-04: 미배정 fallback
  WHERE ae.occurred_at BETWEEN :start AND :end
    AND ae.event_type IN ('message_send', 'workflow_execute')
    -- 디버깅 필터 (H-DASH-03)
    AND (
      (ae.event_type = 'message_send'    AND COALESCE(ae.details->>'invokeFrom', '') != 'debugger')
      OR
      (ae.event_type = 'workflow_execute' AND COALESCE(ae.details->>'triggeredFrom', '') != 'debugging')
    )
    -- AppMode 이중카운트 방어 (H-DASH-01) — collector appMode 보강 후 활성화
    -- AND NOT (ae.event_type = 'workflow_execute' AND ae.details->>'appMode' = 'advanced-chat')
  GROUP BY ad.department_id
)
```

**Step 4: 부서 전체와 LEFT JOIN (활동 없는 부서도 표시)**
```sql
SELECT
  d.id   AS department_id,
  d.name AS department_name,
  COALESCE(dno.new_apps, 0)    AS new_apps,
  COALESCE(dno.new_kbs, 0)     AS new_kbs,
  COALESCE(dno.new_tools, 0)   AS new_tools,
  COALESCE(da.api_calls, 0)    AS api_calls,
  COALESCE(da.token_usage, 0)  AS token_usage
FROM spx_departments d
LEFT JOIN dept_new_objects dno ON dno.department_id = d.id
LEFT JOIN dept_activity da ON da.department_id = d.id
WHERE d.is_active = true

UNION ALL

-- "미배정" 행 (department_id IS NULL)
SELECT
  NULL          AS department_id,
  '미배정'      AS department_name,
  COALESCE((SELECT new_apps  FROM dept_new_objects WHERE department_id IS NULL), 0),
  COALESCE((SELECT new_kbs   FROM dept_new_objects WHERE department_id IS NULL), 0),
  COALESCE((SELECT new_tools FROM dept_new_objects WHERE department_id IS NULL), 0),
  COALESCE((SELECT api_calls FROM dept_activity WHERE department_id IS NULL), 0),
  COALESCE((SELECT token_usage FROM dept_activity WHERE department_id IS NULL), 0)

ORDER BY
  CASE WHEN department_id IS NULL THEN 1 ELSE 0 END,  -- "미배정"은 마지막
  api_calls DESC;                                      -- 디폴트 정렬
```

> **마트 회귀 메모**: 본 쿼리는 OLTP `audit_events` 직접 조회. 추후 `fact_call_daily` 마트로 회귀하면 Step 2의 `audit_events` → `fact_call_daily`로 치환만 하면 됨 (스키마 동일 의도로 설계).

### 서비스 레이어

```python
class DashboardDeptActivityService:
    def get_dept_activity(self, start, end, sort_by="api_calls", sort_order="desc") -> dict:
        """부서별 활동 표 데이터.
        - 신규 App/KB/Tool: spx_resource_ownership 직접 조회 (state 본질)
        - 호출/토큰: audit_events 단일 SoT + spx_resource_ownership LEFT JOIN
        - H-DASH-04: owner_department_id NULL → '미배정' 행으로 합쳐 마지막에 배치
        """
        ...
```

## 프론트엔드 컴포넌트

```
web/app/components/admin/dept-activity/
├── index.tsx                  ← 컨테이너 (URL query string → API 호출)
├── table.tsx                  ← 표 본체 (sticky header + 내부 스크롤)
├── row.tsx                    ← 행 (클릭 없음, hover 시 정확값 tooltip)
└── __tests__/index.spec.tsx
```

별도 훅:
```
web/service/use-dashboard-dept-activity.ts
```

**TanStack Query 훅:**
```typescript
type DeptActivityParams = {
  start: string;
  end: string;
  sortBy?: 'api_calls' | 'token_usage';
  sortOrder?: 'asc' | 'desc';
};

export const useDashboardDeptActivity = (params: DeptActivityParams) =>
  useQuery<DeptActivityResponse>({
    queryKey: ['dashboard', 'dept-activity', params],
    queryFn: () => get('/console/api/dashboard/dept-activity', { params }),
    staleTime: 5 * 60 * 1000,
  });
```

**레이아웃 (고정 높이 + sticky header):**
```tsx
// 외곽: 고정 높이 카드 — 데이터 양과 무관하게 크기 고정
<section className="flex h-[420px] flex-col rounded-xl border bg-components-panel-bg">
  <header className="flex items-center justify-between px-4 py-3">
    <h3 className="text-text-primary">부서별 활동 — {periodLabel}</h3>
  </header>
  <div className="flex-1 overflow-auto">  {/* 본문 스크롤 */}
    <table className="w-full">
      <thead className="sticky top-0 bg-components-panel-bg">
        <tr>
          <th>부서</th>
          <th>신규 App</th>
          <th>신규 KB</th>
          <th>신규 Tool</th>
          <th>호출 수</th>
          <th>토큰 사용</th>
        </tr>
      </thead>
      <tbody>
        {rows.map(r => (
          <tr key={r.department_id ?? 'unassigned'}>
            <td className={r.department_id === null ? 'text-text-tertiary' : ''}>
              {r.department_name}
            </td>
            <td>{r.new_apps || '-'}</td>
            <td>{r.new_kbs || '-'}</td>
            <td>{r.new_tools || '-'}</td>
            <td title={formatExact(r.api_calls)}>{formatCount(r.api_calls)}</td>
            <td title={formatExact(r.token_usage)}>{formatCount(r.token_usage)}</td>
          </tr>
        ))}
      </tbody>
    </table>
  </div>
</section>
```

**인터랙션:**
- 행 클릭 없음 (전역 정책)
- 헤더 정렬/필터: 후보. 전역 표 헤더 정책 확정 시 일괄 적용
- 디자인 토큰: `text-text-*`, `bg-components-*` 사용. raw 색상값 금지

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]]
- [[3. 프로젝트/spx-agent/references/dify-db-schema.md|Dify DB 스키마]]
- [[3. 프로젝트/spx-agent/references/dify-app-modes.md|AppMode 규칙]]
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|RBAC 스키마]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit details 매트릭스]]
