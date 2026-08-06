---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: 차트 드로어 (Phase 2 동적 인터랙션)
phase: 2
status: "🔒 보류 (2026-05-13 이사님)"
harness: []
date: 2026-05-04
last_updated: 2026-05-13
---

> 🔒 **보류 (2026-05-13 이사님 결정)**
>
> 차트 드로어 4종 모두 구현 보류. 좌하 차트 클릭 자체 제거 (차트만 표시).
> 사유:
> 1. 정보 한 단계 깊게 보는 게 사용자에게 의미 있는지 재판단 필요
> 2. 향후 관리자 전용으로 만들 가능성
>
> 본 spec 본문은 재검토 시 base로 활용. 현재 구현 대상 아님.
> 참조: defect-catalog H-DASH-20

# 차트 드로어 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/chart-drawer.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/chart-drawer.md|Tasks]]

## 상태 모델

```typescript
type DrawerMetric = KpiMetric | 'objects-overview';

interface DrawerContext {
  department_id: string | null;     // null = "미배정"
  department_name: string;
  metric: DrawerMetric;
  // period은 별도 prop으로 받음 (drawer state에 박지 않음 — period 변경 시 드로어 닫힘 정책 적용)
}

interface ChartDrawerState {
  context: DrawerContext | null;        // null = 닫힘
  open: (ctx: DrawerContext) => void;
  close: () => void;
  toggle: (ctx: DrawerContext) => void; // 같은 막대 재클릭 → 닫기, 다른 막대 클릭 → 컨텍스트 갱신
}
```

## 훅 설계

```typescript
// use-chart-drawer.ts
export const useChartDrawer = (
  activeMetric: KpiMetric | null,
  period: Period,
): ChartDrawerState => {
  const [context, setContext] = useState<DrawerContext | null>(null);

  const open = useCallback((ctx: DrawerContext) => setContext(ctx), []);
  const close = useCallback(() => setContext(null), []);

  const toggle = useCallback((ctx: DrawerContext) => {
    setContext(prev => {
      if (prev?.department_id === ctx.department_id && prev?.metric === ctx.metric) return null;
      return ctx;
    });
  }, []);

  // 메트릭/기간 변경 시 자동 닫기 (컨텍스트 재구성 회피)
  useEffect(() => {
    if (context) setContext(null);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [activeMetric, period.start, period.end]);

  return { context, open, close, toggle };
};
```

> ESC 핸들러 미설치 — drill-through와 동일 결정 (CLAUDE.md 마운트 환경 규칙).

## 컴포넌트 구조

```
web/app/components/admin/
├── chart-drawer/
│   ├── index.tsx                ← 슬라이드 인 컨테이너 (포털 + 백드롭)
│   ├── drawer-header.tsx        ← 부서명(+팀) + 메트릭 칩 + CSV 버튼 + 닫기 버튼
│   ├── context-chips.tsx        ← 부서/지표/기간/에러 칩 4종 (표시 전용)
│   ├── summary-cards.tsx        ← 호출/사용자 수/에러 3종 (증감률 포함)
│   ├── log-list.tsx             ← 로그 6~10건 + 검색 (클라 필터)
│   ├── more-button.tsx          ← "더 보기 →" 감사 로그 풀뷰 페이지네이션 (placeholder)
│   ├── csv-button.tsx           ← 헤더 내부 사용, 화면만 (백엔드 미연동)
│   ├── use-chart-drawer.ts      ← 컨텍스트 state + 트리거 핸들러
│   ├── use-drawer-data.ts       ← TanStack Query 훅
│   └── types.ts
```

> "전체 보기" 별도 버튼 없음 — "더 보기"가 곧 풀뷰 페이지네이션 진입.

## 컨테이너 (index.tsx) 설계

```typescript
interface Props {
  context: DrawerContext | null;
  period: Period;
  onClose: () => void;
}

export const ChartDrawer: FC<Props> = ({ context, period, onClose }) => {
  // 포커스 트랩 + 드로어 열릴 때 닫기 버튼으로 포커스 이동
  const closeButtonRef = useRef<HTMLButtonElement>(null);
  useEffect(() => {
    if (context) closeButtonRef.current?.focus();
  }, [context]);

  return (
    <Portal>
      {/* 백드롭 */}
      <div
        className={cn(
          'fixed inset-0 z-1003 bg-black/20 transition-opacity duration-200',
          context ? 'opacity-100' : 'pointer-events-none opacity-0',
        )}
        onClick={onClose}
      />
      {/* 슬라이드 인 패널 */}
      <FocusTrap active={!!context}>
        <aside
          role="dialog"
          aria-modal="true"
          aria-labelledby="chart-drawer-title"
          className={cn(
            'fixed right-0 top-0 z-1003 flex h-full w-[480px] flex-col bg-components-panel-bg shadow-xl',
            'transition-transform duration-250',
            context ? 'translate-x-0' : 'translate-x-full',
          )}
        >
          {context && (
            <>
              <DrawerHeader 
                context={context} 
                onClose={onClose} 
                closeButtonRef={closeButtonRef}
              />  {/* CsvButton은 DrawerHeader 안에서 렌더 (우측 영역) */}
              <ContextChips context={context} period={period} />
              <SummaryCards context={context} period={period} />
              <LogList context={context} period={period} />
              <MoreButton context={context} period={period} />
            </>
          )}
        </aside>
      </FocusTrap>
    </Portal>
  );
};
```

> **z-index**: 모달이 `z-1002` (Base UI Dialog 기본). 드로어는 `z-1003`로 모달 위에 별도 레이어. 모달 닫기 버튼/사이드바와 시각 분리.

> **FocusTrap**: Dify가 사용하는 `@base-ui/react`의 `FocusTrap` 또는 `react-focus-lock` 도입. 기존 코드에 패턴이 있으면 따름.

## 트리거 차트 연결

기존 부서별 차트 컴포넌트들에 `onBarClick` prop 추가:

```typescript
// 예: dept-objects-chart/index.tsx
interface Props {
  // ... 기존 필드
  onBarClick?: (department_id: string | null, department_name: string) => void;
}

// ECharts 클릭 이벤트
onEvents={{
  click: (params) => {
    if (onBarClick && params.componentType === 'series') {
      onBarClick(params.data.department_id, params.data.department_name);
    }
  },
}}
```

`DashboardPage`에서:

```typescript
const drawer = useChartDrawer(drillThrough.activeMetric, period);
const handleBarClick = (department_id: string | null, department_name: string) => {
  const metric: DrawerMetric = drillThrough.activeMetric ?? 'objects-overview';
  drawer.toggle({ department_id, department_name, metric });
};

// 종합 뷰
<DeptObjectsChart period={period} onBarClick={handleBarClick} />

// drill-through 활성 모드의 좌 차트
<charts.left period={period} onBarClick={handleBarClick} />
```

## API 엔드포인트

```
GET /console/api/dashboard/drawer
  Query:
    - department_id: string | null  (null = "미배정")
    - metric: objects-overview | calls | users | objects | errors
    - start, end (메트릭 무관 일관 전달)
  Authorization: 로그인 사용자
```

> 단일 엔드포인트로 통일 (드릴 12종 + 드로어 5종 + 기타로 라우터 폭증 방지). 응답 구조가 메트릭에 따라 분기되지만 키는 일관 유지.

## 응답 타입 (TypeScript — 프론트)

```typescript
interface DrawerData {
  context: DrawerContext;            // 에코 (요청 컨텍스트 확인)
  summary: {
    calls: number;                   // 항상 포함 (사용자 기간 따름)
    calls_diff_percent: number | null;     // 이전 기간 대비 증감률 (분모 0이면 null)
    users: number;
    users_diff_percent: number | null;
    error_count: number;             // 실패 호출 절대값 (에러율 % 아님)
    errors_diff_percent: number | null;
  };
  logs: LogEntry[];                  // 6~10건
  has_more: boolean;                 // 더 보기 표시 여부
}

interface LogEntry {
  id: string;
  created_at: string;                // ISO
  user_name: string | null;          // null = 익명/엔드유저
  app_name: string | null;
  app_mode?: string;                 // COMPLETION/CHAT/...
  summary: string;                   // 1~2줄 요약
  has_error: boolean;
}
```

## Response 스키마 (Pydantic — 백엔드)

```python
from datetime import datetime
from pydantic import BaseModel
from app.services.admin.types import DrawerContext  # 또는 같은 모듈에서 정의

class DrawerSummary(BaseModel):
    calls: int                              # 사용자 기간 따름
    calls_diff_percent: float | None = None # 이전 기간 대비, 분모 0이면 None (H-DASH-07)
    users: int
    users_diff_percent: float | None = None
    error_count: int                        # 실패 호출 절대값
    errors_diff_percent: float | None = None

class LogEntry(BaseModel):
    id: str
    created_at: datetime            # FastAPI/Flask 직렬화 시 ISO 자동
    user_name: str | None = None    # None = 익명/엔드유저
    app_name: str | None = None
    app_mode: str | None = None     # COMPLETION/CHAT/...
    summary: str                    # 1~2줄 요약
    has_error: bool

class DrawerData(BaseModel):
    context: DrawerContext          # 요청 에코
    summary: DrawerSummary
    logs: list[LogEntry]            # 6~10건
    has_more: bool                  # "더 보기" 노출 여부
```

> **datetime 직렬화**: `model_dump(mode='json')` 사용 시 ISO 문자열 자동 변환. Flask 응답에서는 `jsonify(data.model_dump(mode='json'))` 패턴.

## 쿼리 설계 (백엔드 spec)

> **승랑님 스키마 의존**. 받기 전엔 mock 응답으로 진행.

**요약 카드 3종 (재사용 + 증감률)**:
- `calls` + `calls_diff_percent`: KPI 카드 `get_api_calls()`에 `WHERE department_id = :department_id` 추가, 현재/이전 기간 두 번 쿼리 후 증감률 계산
- `users` + `users_diff_percent`: KPI 카드 `get_users()`에 부서 필터 + 이전 기간 비교
- `error_count` + `errors_diff_percent`: 실패 호출 **절대값** (에러율 % 아님). `messages.status='error'` + `workflow_runs.status='failed'` 카운트, 이전 기간 대비 증감
- **모두 사용자 period를 따름**
- 증감률 분모 0이면 `None` 반환 (H-DASH-05/H-DASH-07)

**로그 6~10건**:
```sql
-- COMPLETION/CHAT/AGENT_CHAT/ADVANCED_CHAT
SELECT m.id, m.created_at, m.from_account_id, m.app_id, a.name AS app_name, a.mode,
       LEFT(m.query, 80) AS summary, m.error IS NOT NULL AS has_error
FROM messages m
JOIN apps a ON m.app_id = a.id
LEFT JOIN spx_resource_ownership o ON o.resource_id = a.id AND o.resource_type = 'app'
WHERE m.created_at BETWEEN :start AND :end
  AND m.invoke_from != 'debugger'
  AND (o.department_id = :department_id OR (:department_id IS NULL AND o.id IS NULL))
ORDER BY m.created_at DESC
LIMIT 10;

-- workflow_runs도 동일 패턴 (status='failed' / triggered_from='app-run')
-- 두 결과 UNION ALL → 시각 정렬 → LIMIT 10
```

> 미배정 처리: `department_id IS NULL` 시 `spx_resource_ownership.id IS NULL` 행 (H-DASH-04 fallback).

**증감/이전 기간 사용**: 요약 카드 3종 모두 이전 기간 대비 증감률 표시 (`calc_diff()` 호출). 메인 KPI와 별개로 부서 컨텍스트 안에서의 증감을 보여줌.

## TanStack Query 훅

```typescript
// use-drawer-data.ts
export const useDrawerData = (
  context: DrawerContext | null,
  period: Period,
) => {
  return useQuery<DrawerData>({
    queryKey: ['dashboard', 'drawer', context?.department_id, context?.metric, period],
    queryFn: () => get('/dashboard/drawer', {
      params: {
        department_id: context!.department_id,
        metric: context!.metric,
        start: period.start,
        end: period.end,
      },
    }),
    enabled: !!context,
    staleTime: 5 * 60 * 1000,
  });
};
```

- `enabled: !!context` → 닫힌 상태에서 fetch 안 함
- queryKey에 `department_id` 포함 → 다른 부서로 컨텍스트 변경 시 별도 캐시
- 5분 staleTime → 같은 부서 재진입 시 즉시 표시

## CSV 버튼 (화면만, 헤더 우측)

```typescript
// drawer-header.tsx 내부에서 렌더 (별도 헤더 영역 아님)
export const CsvButton: FC<{ context: DrawerContext }> = () => {
  const handleClick = () => {
    toast.info('CSV 다운로드는 추후 지원 예정입니다');  // 백엔드 미구현
  };
  return (
    <Button variant="ghost" size="sm" onClick={handleClick} title="CSV 내보내기">
      <RiDownloadLine className="size-4" />
    </Button>
  );
};
```

> 위치: 헤더 우측 (닫기 버튼 옆). 본문 영역 차지 없음 — 아이콘 버튼만.
> 또는 `disabled` + 툴팁 ("백엔드 미구현")으로 처리 가능. UX 검증 후 결정.

## 더 보기 버튼 (감사 로그 풀뷰 페이지네이션)

```typescript
// more-button.tsx
export const MoreButton: FC<{ context: DrawerContext; period: Period }> = ({ context, period }) => {
  const handleClick = () => {
    // Phase 2 후속 — 감사 로그 풀뷰 페이지 미구현
    toast.info('감사 로그 풀뷰는 추후 지원 예정입니다');
    // 구현 시 흐름:
    // const params = new URLSearchParams({ 
    //   department_id: context.department_id ?? '',
    //   metric: context.metric, 
    //   start: period.start, end: period.end, 
    //   error: context.errorOnly ? '1' : '0',
    // });
    // router.push(`/dashboard/audit-logs?${params}`);
  };
  return (
    <Button variant="ghost" onClick={handleClick} className="w-full">
      더 보기 <RiArrowRightLine className="ml-1 size-4" />
    </Button>
  );
};
```

> 클릭 시 현재 드로어 컨텍스트(부서/메트릭/기간/에러 필터) 그대로 풀뷰 페이지에 전달.
> Phase 2 본 spec에서는 placeholder 토스트로 처리. 풀뷰 페이지는 후속 spec.

## 디자인 토큰

| 용도 | 토큰 | Tailwind |
|---|---|---|
| 드로어 배경 | `--color-components-panel-bg` | `bg-components-panel-bg` |
| 드로어 그림자 | (시스템 그림자) | `shadow-xl` |
| 백드롭 | (검정 20%) | `bg-black/20` |
| 헤더 구분선 | `--color-divider-burn` | `border-divider-burn` |
| 메트릭 칩 | `--color-components-badge-*` | (기존 칩 패턴) |
| 검색 입력 | (기존 SearchInput 컴포넌트) | — |

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md|drill-through design]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/dept-objects.md|dept-objects design]]
