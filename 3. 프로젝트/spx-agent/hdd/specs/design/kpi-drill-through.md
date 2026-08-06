---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: KPI 드릴스루 (Phase 2 동적 인터랙션)
phase: 2
harness: [H-DASH-04, H-DASH-07, H-DASH-18, H-DASH-19, H-DASH-20]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_pdf: "0. Inbox/화면설계_v0.3.pdf"
reference_pdf_pages: "17"
date: 2026-05-04
last_updated: 2026-06-09
ui_alignment: 2026-05-22
---
# KPI 드릴스루 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/kpi-drill-through.md|Tasks]]

## 상태 모델

```typescript
type KpiMetric = 'objects' | 'users' | 'calls' | 'apps';

// URL query string이 단일 진실: ?metric=objects (없으면 종합 뷰)
const activeMetric: KpiMetric | null = parseMetricFromQuery(searchParams);
```

- 소유자: `web/app/(commonLayout)/dashboard/page.tsx` (Next.js `useSearchParams`)
- 전파 경로: prop으로 `KpiSection` + 차트/표 영역 컨테이너에 전달
- 토글: `router.replace('?metric=' + m)` (active면 `router.replace('?')`로 해제)
- Deep link 자연 지원 — `/dashboard?metric=calls`로 진입 가능

## 훅 설계

```typescript
// use-kpi-drill-through.ts
export const useKpiDrillThrough = () => {
  const searchParams = useSearchParams();
  const router = useRouter();
  const activeMetric = (searchParams.get('metric') as KpiMetric) ?? null;

  const toggle = useCallback((m: KpiMetric) => {
    const params = new URLSearchParams(searchParams);
    if (activeMetric === m) params.delete('metric');
    else params.set('metric', m);
    router.replace(`?${params.toString()}`);
  }, [activeMetric, searchParams, router]);

  // ESC: drill-through 해제 (다른 닫기 대상이 없을 때만)
  useEffect(() => {
    if (!activeMetric) return;
    const onKey = (e: KeyboardEvent) => {
      if (e.key !== 'Escape') return;
      if (e.defaultPrevented) return; // 다른 핸들러가 우선
      toggle(activeMetric);
    };
    window.addEventListener('keydown', onKey);
    return () => window.removeEventListener('keydown', onKey);
  }, [activeMetric, toggle]);

  return { activeMetric, toggle };
};
```

## 영역 매핑 (PDF v0.3 17p 기준)

> 차트·표의 위치 / 영역 높이 / 색상 체계는 메트릭과 무관하게 유지. 좌하·우하 차트는 표시 전용(클릭 인터랙션 없음 — H-DASH-20).

| 메트릭 | 좌하 (가로 막대) | 우하 (가로 막대) | 하단 (표) |
|---|---|---|---|
| **objects** | 부서별 누적 오브젝트 | Top 소유자 (이름·부서·수치) | 부서별 신규 생성 (5컬럼) |
| **users** | 부서별 앱 이용 수 | 앱 이용자 Top10 | 부서별 앱 이용 현황 (5컬럼) |
| **calls** | 부서별 앱 호출 수 | 모델별 호출 분포 | 부서별 앱 호출 현황 (5컬럼) |
| **apps** | 호출 수 Top10 | 에러 발생 Top10 | 앱별 이용 현황 (6컬럼) |

> KPI 4(`apps`) 활성 시 보조 카드 신설 없음 — KPI 4 카드 자체가 active로 강조(앱명 큰 자리 + 호출 수 회색 배지) + 좌하·우하·하단만 교체.

## 컴포넌트 구조

```
web/app/components/admin/
├── kpi-section/
│   ├── index.tsx                ← KPI 4개 그리드 (기존)
│   ├── kpi-card.tsx             ← isActive / metricKey / onActivate prop
│   └── ...
├── dashboard-page/
│   ├── index.tsx                ← useKpiDrillThrough 소유, activeMetric 전파
│   └── chart-area.tsx           ← activeMetric 분기, 좌하 / 우하 / 하단 슬롯 교체
├── drill-charts/
│   ├── objects/
│   │   ├── dept-cumulative.tsx        ← 좌하: 부서별 누적
│   │   └── top-owners.tsx             ← 우하: Top 소유자
│   ├── users/
│   │   ├── dept-adopted-apps.tsx      ← 좌하: 부서별 앱 이용 수
│   │   └── top-users.tsx              ← 우하: 앱 이용자 Top10
│   ├── calls/
│   │   ├── dept-call-count.tsx        ← 좌하: 부서별 호출 수
│   │   └── model-call-share.tsx       ← 우하: 모델별 호출 분포 (5/20 dept-error-rate 대체 옵션 폐기 — dead code 청산)
│   └── apps/
│       ├── app-call-top10.tsx         ← 좌하: 호출 수 상위 10개 앱
│       └── top-error-apps.tsx         ← 우하: 에러율 상위 10개 앱
└── drill-tables/
    ├── dept-new-creations-table.tsx   ← objects 하단
    ├── dept-activity-table.tsx        ← users 하단
    ├── dept-call-rps-table.tsx        ← calls 하단 (5/20 명명 정합 — RPS 컬럼 강조)
    └── app-stats-table.tsx            ← apps 하단 (행 클릭 → /app/{appId}/overview)
```

## chart-area.tsx 설계

```typescript
interface Props {
  activeMetric: KpiMetric | null;
  period: Period;
}

interface DrillSlots {
  Left: FC<{ period: Period }>;
  Right: FC<{ period: Period }>;
  BottomTable: FC<{ period: Period }>;
}

const DRILL_SLOT_MAP: Record<KpiMetric, DrillSlots> = {
  objects: { Left: DeptCumulative,   Right: TopOwners,      BottomTable: DeptNewCreationsTable },
  users:   { Left: DeptAdoptedApps,  Right: TopUsers,       BottomTable: DeptActivityTable },
  calls:   { Left: DeptCallCount,    Right: ModelCallShare, BottomTable: DeptCallRpsTable },
  apps:    { Left: AppCallTop10,     Right: TopErrorApps,   BottomTable: AppStatsTable },
};

export const ChartArea: FC<Props> = ({ activeMetric, period }) => {
  if (activeMetric === null) {
    return (
      <>
        <DeptObjectsChart period={period} />
        <ModelTokensChart period={period} />
        <DeptActivityTable period={period} />
      </>
    );
  }
  const slots = DRILL_SLOT_MAP[activeMetric];
  return (
    <div className="transition-opacity duration-200" key={activeMetric}>
      <slots.Left period={period} />
      <slots.Right period={period} />
      <slots.BottomTable period={period} />
    </div>
  );
};
```

## 표 4종 컬럼 명세

> 모든 표는 행 단위 = 부서 (apps 표만 행 단위 = 앱). 미배정 부서 행 유지 (H-DASH-04). 영역 고정 높이 + 내부 스크롤(5/13 전역).

### objects: 부서별 신규 생성 (`DeptNewCreationsTable`)

| 컬럼 | 타입 | 비고 |
|---|---|---|
| 부서 | string | 부서명 (미배정 포함) |
| App | int | 신규 생성 수 (`+` prefix) |
| KB | int | 신규 생성 수 |
| Tool | int | 신규 생성 수 |
| 신규 | int | 합계 (강조 색상) |

> ⚠️ PM 확인 — "신규"의 시점 정의: `resource_ownership` 등록 시점 vs Dify 앱 / KB / Tool 생성 시점

행 클릭: **없음**.

### users: 부서별 앱 이용 현황 (`DeptUsersTable`)

> 파일명: `drill-tables/dept-users-table.tsx` (5/22 코드 정합).

| 컬럼 | 타입 | 비고 |
|---|---|---|
| 부서 | string | `actor_dept_id → spx_departments.name` (미배정 포함, H-DASH-04). 부서명 가나다순 기본 정렬 |
| 부서원 | int | `COUNT(DISTINCT actor_id)` WHERE `actor_dept_id = dept` |
| 부서원당 이용 앱 수 | float | `COUNT(DISTINCT target_app_id) / NULLIF(COUNT(DISTINCT actor_id), 0)`. 1자리 소수점 (예: `0.5`, `2.3`). 분모 0 → "-" |
| Top 이용 앱 | string | 부서 내 호출 수 1위 앱명. `DISTINCT ON (actor_dept_id) ... ORDER BY actor_dept_id, COUNT(*) DESC` + `apps.name` JOIN |
| 추세 | badge | 이용 앱 수의 전기간 대비 증감 % (KPI 카드 증감 뱃지 패턴). 분모 0 → "+신규" / "-" 분기 (H-DASH-07) |

행 클릭: **없음** (전역 정책 — KPI 4만).

**좌하 차트와의 정보 분리**: 좌하 = 부서별 앱 이용 수 막대 (절대값). 표는 부서원 수 base + 부서원당 이용 앱 수 정규화 + Top 이용 앱 (질적) + 추세 (시간 차원)로 보완 — 정보 중복 회피.

**백엔드 단일 CTE 쿼리** (5/19 부하 테스트 dept-user-activity 5.8초 병목 = 7쿼리 순차 실행 원인 해소):

```sql
WITH base AS (
  SELECT actor_dept_id, actor_id, target_app_id
  FROM spx_mv_audit_enriched
  WHERE tenant_id = :tenant
    AND occurred_at BETWEEN :start AND :end
),
agg AS (
  SELECT
    actor_dept_id,
    COUNT(DISTINCT actor_id) AS user_count,
    COUNT(DISTINCT target_app_id) AS adopted_apps
  FROM base
  GROUP BY actor_dept_id
),
top_app AS (
  SELECT DISTINCT ON (actor_dept_id)
    actor_dept_id, target_app_id
  FROM base
  GROUP BY actor_dept_id, target_app_id
  ORDER BY actor_dept_id, COUNT(*) DESC
),
prev_agg AS (
  SELECT actor_dept_id, COUNT(DISTINCT target_app_id) AS prev_adopted
  FROM spx_mv_audit_enriched
  WHERE tenant_id = :tenant
    AND occurred_at BETWEEN :prev_start AND :prev_end
  GROUP BY actor_dept_id
)
SELECT
  COALESCE(d.name, '미배정')                                                AS dept_name,
  a.user_count                                                              AS user_count,
  ROUND(a.adopted_apps::numeric / NULLIF(a.user_count, 0), 1)               AS apps_per_user,
  app.name                                                                  AS top_app_name,
  (a.adopted_apps - COALESCE(pa.prev_adopted, 0))::float
    / NULLIF(pa.prev_adopted, 0) * 100                                      AS trend_pct
FROM agg a
LEFT JOIN spx_departments d  ON d.id = a.actor_dept_id
LEFT JOIN top_app ta         ON ta.actor_dept_id = a.actor_dept_id
LEFT JOIN apps app           ON app.id = ta.target_app_id
LEFT JOIN prev_agg pa        ON pa.actor_dept_id = a.actor_dept_id
ORDER BY a.user_count DESC;
```

- 7쿼리 → 1쿼리. NEW/CHURNED NOT IN 서브쿼리 자체가 사라져 부하 테스트 dept-user-activity p95 5.8s → 1s 이하 자연 해소 예상
- `spx_mv_audit_enriched` 인덱스 hit (`tenant_id, occurred_at`) 유지 — 5/20 추가 인덱스 효과 그대로
- `apps` JOIN은 Top 채택 앱 1행/부서만 필요해서 비용 작음

### calls: 부서별 앱 호출 현황 (`DeptCallRpsTable`)

| 컬럼 | 타입 | 비고 |
|---|---|---|
| 부서 | string | 부서명 (미배정 / 외부 포함). 부서 기준 = `app_owner_dept_id` (앱 소유 부서). 부서명 가나다순 기본 정렬 |
| 호출 | int | 기간 내 호출 수 = `SUM(spx_mv_kpi_calls_daily.calls)` (K/M 압축, 사용자 period) |
| RPS | float | 기간 평균 RPS = `SUM(calls) / period_seconds`. 소수점 2자리. 분모 = `max((end - start) 초, 1)` |
| Top 호출 앱 | string | 부서 소유 앱 중 호출 1위. `ROW_NUMBER() OVER (PARTITION BY app_owner_dept_id ORDER BY SUM(calls) DESC) = 1` + `apps.name` JOIN. 앱 삭제 시 `"삭제된 앱"` fallback |
| 추세 | badge | 호출 수 전기간 대비 증감 %. 분모 0 → "-" / null (H-DASH-07) |

행 클릭: **없음** (전역 정책 — KPI 4만).

**좌하 차트와의 정보 분리**: 좌하 = 부서별 호출 수 막대 (절대값). 표는 RPS(기간 정규화) + Top 앱(질적) + 추세(시간 차원)로 보완 — 정보 중복 회피.

**백엔드 쿼리 패턴** — Layer 2 `spx_mv_kpi_calls_daily` enforce:

```sql
-- 1) 현 기간 부서별 합산 (`get_dept_call_count` 재사용 + curr)
SELECT app_owner_dept_id, SUM(calls) AS calls
FROM spx_mv_kpi_calls_daily
WHERE tenant_id = :tenant AND day BETWEEN :start AND :end
GROUP BY app_owner_dept_id;

-- 2) 이전 기간 (추세 산출용)
SELECT app_owner_dept_id, SUM(calls) AS calls
FROM spx_mv_kpi_calls_daily
WHERE tenant_id = :tenant AND day BETWEEN :prev_start AND :prev_end
GROUP BY app_owner_dept_id;

-- 3) Top 앱 (window function, rn=1만 추출)
SELECT app_owner_dept_id, COALESCE(apps.name, '삭제된 앱') AS app_name
FROM (
  SELECT app_owner_dept_id, target_app_id,
         ROW_NUMBER() OVER (PARTITION BY app_owner_dept_id ORDER BY SUM(calls) DESC) AS rn
  FROM spx_mv_kpi_calls_daily
  WHERE tenant_id = :tenant AND day BETWEEN :start AND :end
  GROUP BY app_owner_dept_id, target_app_id
) t LEFT JOIN apps ON apps.id = t.target_app_id
WHERE rn = 1;

-- 4) Department.name lookup (UNASSIGNED / EXTERNAL sentinel 제외)
```

- 3쿼리 분리 패턴 (curr / prev / top_app + dept name lookup) — N+1 아님 (부서 수만큼 반복하지 않음)
- 단일 CTE 통합도 가능하지만, Layer 2 mart는 row 수가 작아(부서×앱×day) 분리 쿼리가 명료
- 추세 계산 = `(calls - prev) / prev * 100`. prev=0 → null (프론트 "-" 분기, H-DASH-07)
- 미배정/외부 sentinel: `UNASSIGNED_DEPT_ID` / `EXTERNAL_DEPT_ID` 상수 (models/mart.py)

> ⚠️ PM 확인 — api_call (nginx) 부서별 분류 가능성. 현재 좌하 차트·표는 audit 마트 기반(콘솔 + end_user)만 카운트. nginx 외부 API key 호출 분류는 collector 보강 후 별도 검토.

### apps: 앱별 이용 현황 (`AppStatsTable`) — KPI 4 전용

| 컬럼 | 타입 | 비고 |
|---|---|---|
| 앱 | string | 앱 이름. 우상단에 모드 뱃지(Chat / Workflow / Completion) |
| 부서 | string | 앱 owner_dept (미배정 fallback) |
| 호출 | int | 기간 내 호출 수 |
| 이용자 | int | `COUNT(DISTINCT actor_id)` |
| 에러 | float (%) | ILIKE 6종 분류 합계 ÷ 호출 (소수점 1자리) |
| 마지막 사용 | relative time | `MAX(occurred_at)` |

기본 정렬: 호출 내림차순(서버 응답 순서 그대로). 부서명 가나다순 미적용 (앱 그레인).

행 클릭: **같은 탭에서 `/app/{appId}/overview`** (Dify 모니터링 페이지). `role="link"` + `tabIndex={0}` + 키보드 Enter 지원.

> 선택 후보 컬럼 (응답시간 / 만족도 / TPS / 앱 타입 컬럼): 본 단계 검증 작업 없음. PM 결정 시 audit 가용성 확인 후 추가.

**행 grain = 앱** (다른 표 3종은 모두 부서 grain). KPI 3 `DeptCallTable`(부서 grain)과 명확히 구분.

**백엔드 단일 쿼리** (앱 단위 GROUP BY + owner 부서 / 앱명 JOIN):

```sql
SELECT
  app.id                                                          AS app_id,
  app.name                                                        AS app_name,
  COALESCE(d.name, '미배정')                                      AS owner_dept_name,
  COUNT(*)                                                        AS calls,
  COUNT(DISTINCT ae.actor_id)                                     AS users,
  ROUND(
    COUNT(*) FILTER (WHERE ae.details->>'error' ILIKE ANY (ARRAY[
      '%rate limit%', '%timeout%', '%unauthorized%',
      '%quota%', '%model error%', '%'
    ]))::numeric / NULLIF(COUNT(*), 0) * 100,
    1
  )                                                               AS error_rate,
  MAX(ae.occurred_at)                                             AS last_used
FROM spx_mv_audit_enriched ae
JOIN apps app                       ON app.id = ae.target_app_id
LEFT JOIN spx_resource_ownership ro ON ro.resource_id = ae.target_app_id AND ro.resource_type = 'app'
LEFT JOIN spx_departments d         ON d.id = ro.owner_department_id
WHERE ae.tenant_id = :tenant
  AND ae.occurred_at BETWEEN :start AND :end
GROUP BY app.id, app.name, d.name
ORDER BY calls DESC
LIMIT 100;
```

- 좌하 차트 (`AppCallTop10` — "호출 수 상위 10개 앱"): 위 쿼리에 `LIMIT 10`만 적용
- 우하 차트 (`TopErrorApps` — "에러율 상위 10개 앱"): `ORDER BY error_count DESC LIMIT 10`로 변형 (ILIKE 6종 필터링 결과 기준)
- `LIMIT 100`은 앱 폭증 대응 (페이지네이션 도입 시점 PM 결정 — 본 단계 100 fixed)

### 공통 표 동작

- 정렬: 메트릭 의미상 큰 컬럼 기본 내림차순 (objects=신규, users=사용자 수, calls=호출, apps=호출)
- 미배정 부서 / 미배정 앱 행: 항상 마지막 또는 별도 강조 (H-DASH-04)
- 표 헤더 정렬·필터: **글로벌 정책 후보** (전체 표 공통 결정 후 적용, 본 spec 단독 결정 X)
- 합계 정합성: 표의 컬럼 합계 = 좌하 막대 차트 합계 = 활성 KPI 카드 수치
- 디자인 토큰: 종합 뷰의 `dept-activity-table`과 동일 토큰

## KPI 카드 확장

```typescript
interface KpiCardProps {
  // ... 기존 필드
  isActive?: boolean;
  isClickable?: boolean;
  metricKey?: KpiMetric;
  onActivate?: (m: KpiMetric) => void;
}

// 시각 강조 (isActive 시) — 배경 토큰 변경 없음:
// cn(
//   baseCardClass,
//   isActive && 'border-transparent ring-inset ring-2 ring-components-button-primary-bg-hover shadow-lg',
//   isClickable && !isActive && 'cursor-pointer hover:shadow-lg',
// )
// role="button" tabIndex={0} aria-pressed={isActive}
// onKeyDown: Enter/Space → onActivate(metricKey)
```

KPI 4 활성 시 KPI 4 카드 자체가 active로 강조 — 큰 자리에 앱명(subtitle truncate + hover), 뱃지 자리에 호출 수(회색 배지). 별도 보조 카드 신설 없음.

## top-users 쿼리 필터 (5/22 정합)

`api/services/admin/dashboard_drill_users_service.py:_TOP_USERS_SQL` — 디버그/비정규 호출 제외해 진짜 앱 이용만 카운트:

```sql
SELECT
  COALESCE(a.name, '알 수 없음')        AS user_name,
  COALESCE(d.name, '미배정')            AS department_name,
  COUNT(*)                              AS activity_count
FROM spx_mv_audit_enriched ae
JOIN spx_accounts a ON a.id = ae.actor_id
LEFT JOIN spx_department_members dm ON dm.account_id = ae.actor_id
LEFT JOIN spx_departments d ON d.id = dm.department_id
WHERE ae.tenant_id = :tenant
  AND ae.occurred_at BETWEEN :start AND :end
  AND ae.actor_id IS NOT NULL
  AND NOT ae.is_debug              -- 디버그 호출 제외 (H-DASH-03)
  AND ae.is_canonical_call         -- 정규 호출만 (H-DASH-01 합성)
GROUP BY ae.actor_id, a.name, d.name
ORDER BY activity_count DESC
LIMIT 10
```

- 필터 2종(`NOT is_debug` / `is_canonical_call`)으로 (a) 디버깅 실행, (b) ADVANCED_CHAT의 workflow_runs 이중카운트, 두 결함을 audit 마트 derived 컬럼 단위에서 일괄 차단
- 적용 위치: 우하 차트 "앱 이용자 Top10" 데이터 산출

## 데이터 소스 — 마트 입력

> audit_events 단일 SoT + RBAC JOIN이 기본. 오브젝트만 `spx_resource_ownership` 직접 조회.

### 공통 JOIN 패턴

```sql
-- 사용자 / 호출 / 에러 메트릭 (audit_events 기반)
SELECT
  COALESCE(d.name, '미배정') AS bucket,
  COUNT(*) AS calls,
  COUNT(DISTINCT ae.actor_id) AS users,
  COUNT(*) FILTER (
    WHERE ae.details->>'error' ILIKE ANY (ARRAY[
      '%rate limit%', '%timeout%', '%unauthorized%',
      '%quota%', '%model error%', '%'
    ])
  ) AS errors,
  MAX(ae.occurred_at) AS last_used
FROM audit_events ae
LEFT JOIN spx_resource_ownership ro ON ro.resource_id = ae.app_id AND ro.resource_type = 'app'
LEFT JOIN spx_departments d ON d.id = ro.department_id
WHERE ae.occurred_at BETWEEN :start AND :end
  AND ae.details->>'invokeFrom' != 'debugger'
  AND ae.details->>'triggeredFrom' != 'debugging'
GROUP BY bucket;
```

- `ae.actor_type = 'end_user'`로 외부 사용자 식별 가능
- 부서 매핑: `spx_department_members` JOIN으로 actor → 부서 (users 메트릭)
- 미배정: LEFT JOIN의 NULL → "미배정" fallback (H-DASH-04)
- 디버깅 필터: audit `details` 보강 후 적용. 미보강 상태에서는 후보 Harness `H-CAND-audit-appmode-missing`, `H-CAND-audit-wf-debug-filter-missing` 적용

### objects 전용 — `spx_resource_ownership` 직접 조회

```sql
SELECT
  COALESCE(d.name, '미배정') AS bucket,
  COUNT(*) FILTER (WHERE ro.resource_type = 'app')  AS app_count,
  COUNT(*) FILTER (WHERE ro.resource_type = 'kb')   AS kb_count,
  COUNT(*) FILTER (WHERE ro.resource_type = 'tool') AS tool_count
FROM spx_resource_ownership ro
LEFT JOIN spx_departments d ON d.id = ro.department_id
GROUP BY bucket;
```

- audit 본질 불가(오브젝트는 state) — `spx_resource_ownership` 단독 조회
- 부서 차원으로 누적량 / 신규 생성 / Top 소유자 도출

## API 엔드포인트

```
GET /console/api/dashboard/drill/{metric}/{chart}
  Path:
    metric: objects | users | calls | apps   (5/13 결정 — errors → apps로 4번째 metric 변경)
    chart: dept-cumulative | top-owners | ... (메트릭별 유효 차트만)
  Query:
    start, end (사용자 period)
  Authorization: 로그인 사용자 (AppInitializer 가드만)
```

총 11개 엔드포인트:

| metric | charts |
|---|---|
| `objects` | `dept-cumulative` / `top-owners` / `dept-new-creations` |
| `users`   | `dept-adopted-apps` / `top-users` / `dept-activity` |
| `calls`   | `dept-call-count` / `model-call-share` / `dept-call` |
| `apps`    | `app-call-top10` / `top-error-apps` / `app-stats` |

### 응답 스키마 — 표 4종 (5/20 신설 — users/apps, calls/objects 5/20 후속 박음)

> **2026-06-09 id-키잉**: 부서 그레인 표/차트는 `department_id`를 응답에 포함 + **프론트 React key·dedup 기준 = `department_id`**(name은 PK 아님, 동명 부서 존재). dept 그레인 차트(`DeptCallCount`/`DeptAdoptedApps`/`DeptCumulative`)도 동일하게 `department_id` 포함. `department_name`은 표시용.

```typescript
// GET /console/api/dashboard/drill/objects/dept-new-creations?start=...&end=...
type DrillDeptNewCreationsResponse = {
  items: Array<{
    department_id: string;    // React key·dedup 기준 (2026-06-09 id-키잉)
    department_name: string;  // "미배정" 포함 (표시용)
    app_count: number;
    kb_count: number;
    tool_count: number;
    total_new: number;
  }>;
};

// GET /console/api/dashboard/drill/users/dept-activity?start=...&end=...
type DeptActivityResponse = {
  rows: Array<{
    department_id: string;         // React key·dedup 기준 (2026-06-09 id-키잉)
    dept_name: string;             // "미배정" 포함 (표시용)
    user_count: number;
    apps_per_user: number | null;  // 분모(사용자 수) 0 시 null → 프론트 "-"
    top_app_name: string | null;   // 활동 없으면 null
    trend_pct: number | null;      // 이전 기간 0 시 null
    trend_label?: '+신규' | '-';   // null 분기 라벨 (H-DASH-07)
  }>;
};

// GET /console/api/dashboard/drill/calls/dept-call-rps-table?start=...&end=...
type DrillDeptCallRpsResponse = {
  items: Array<{
    department_id: string;         // React key·dedup 기준 (2026-06-09 id-키잉)
    department_name: string;       // "미배정" / "외부" 포함 (표시용)
    calls: number;
    rps: number;                   // 소수점 2자리
    top_app_name: string | null;   // 활동 없으면 null. 앱 삭제 시 "삭제된 앱"
    trend_percent: number | null;  // 이전 기간 0 시 null → 프론트 "-"
  }>;
};

// GET /console/api/dashboard/drill/apps/app-stats?start=...&end=...
type AppStatsResponse = {
  rows: Array<{
    app_id: string;          // 행 클릭 → /app/{app_id}/overview
    app_name: string;
    owner_dept_name: string; // "미배정" 포함
    calls: number;
    users: number;
    error_rate: number | null;  // 호출 0 시 null → 프론트 "N/A"
    last_used: string;       // ISO 8601
  }>;
};
```

> 응답 키 표준: `items` (objects/calls) vs `rows` (users/apps) — 5/20 KPI 2/4 정합 시 신설된 표는 `rows`, 기존 잔존은 `items`. 추후 통일 검토 가능하나 본 위임 범위 외.

> KPI 4(`apps`) 활성 모드의 좌하·우하·하단 데이터는 `errors` 차원(2종) + `calls.dept-call-count` 일부 재활용. URL 경로의 `metric=errors`는 KPI 4의 표·차트 dimension을 표현하기 위한 분리(전체 `apps` 디멘션이 아니라 에러 중심 슬라이스). KPI 4 표는 `errors.app-stats` 엔드포인트에서 통합 조회.

대안으로 통합 엔드포인트(`GET /console/api/dashboard/drill?metric=&chart=&start=&end=`)도 가능 — 라우터 정리는 백엔드 작업 시 결정.

## 데이터 페칭

```typescript
const useDrillChart = (metric: KpiMetric, chart: string, params: { start: string; end: string }) => {
  return useQuery({
    queryKey: ['dashboard', 'drill', metric, chart, params],
    queryFn: () => get(`/console/api/dashboard/drill/${metric}/${chart}`, { params }),
    staleTime: 5 * 60 * 1000,
    enabled: !!params.start && !!params.end,
  });
};
```

- queryKey 접두사 `['dashboard']` — 페이지 헤더 새로고침이 일괄 invalidate
- 메트릭 전환 시 캐시 유지 → 재진입 즉시 표시
- 페이지 헤더의 기간 변경 시 staleTime 무시하고 refetch (페이지 단위 query 표준)

## 디자인 토큰

| 용도 | 토큰 | Tailwind |
|------|------|---------|
| Active 카드 ring | `--color-components-button-primary-bg-hover` | `ring-inset ring-2 ring-components-button-primary-bg-hover` |
| Active 카드 배경 | (변경 없음 — 기본 `bg-components-card-bg` 유지) | — |
| 비active 호버 | (shadow 강조) | `hover:shadow-lg` |
| 차트/표 컨테이너 | `--color-components-chart-bg` | `bg-components-chart-bg` |
| 컬러 체계 (App/KB/Tool) | `--util-colors-blue-*` / `--util-colors-teal-*` / `--util-colors-orange-*` | `util-colors-blue-blue-500` etc. |

## 전환 애니메이션

```tsx
<div className="transition-opacity duration-200" key={activeMetric ?? 'overview'}>
  {/* 차트들 */}
</div>
```

`key`를 바꾸면 React가 언마운트/리마운트 → 페이드 효과.

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md|KPI 카드 design]]
- [[3. 프로젝트/spx-agent/references/dify-db-schema.md|Dify DB 스키마]]
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|RBAC 스키마]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit details 가용성]]
