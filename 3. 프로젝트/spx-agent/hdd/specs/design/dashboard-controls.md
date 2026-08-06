---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: 대시보드 컨트롤
harness: []
date: 2026-04-30
last_updated: 2026-05-13
---
# 대시보드 컨트롤 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dashboard-controls.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dashboard-controls.md|Tasks]]

## 컴포넌트 구조

```
web/app/components/admin/dashboard-controls/
├── index.tsx              ← 컨테이너 (기간/DatePicker/새로고침 가로 배치)
├── time-range-picker.tsx  ← SimpleSelect 기반 5개 옵션 드롭다운
├── date-picker.tsx        ← 사용자 지정 시작/종료일 (custom 선택 시)
├── refresh-button.tsx     ← 새로고침 버튼
├── use-period-query.ts    ← URL query string 기반 period 훅
├── constants.ts           ← TIME_PERIOD_OPTIONS (5개), DEFAULT_PERIOD_KEY, QUERY_DATE_FORMAT
└── types.ts               ← Period, TimePeriodKey
```

> 폴더명 `admin/`은 유지 (2026-05-13 — `dashboard/`로 통일 검토는 별건). 향후 통일 시 본 spec 폴더 경로만 일괄 치환.

## 마운트 통합 — 페이지 우상단 단독 배치 (2026-05-14 갱신)

대시보드 페이지(`app/(commonLayout)/dashboard/page.tsx`) 자체 `<h1>` 미작성 (톱 네비 활성 상태로 충분, 스튜디오/지식 페이지와 동일 패턴). `dashboard-controls`는 페이지 상단 우측에 단독 정렬.

```tsx
// app/(commonLayout)/dashboard/page.tsx
'use client'
import DashboardControls from '@/app/components/admin/dashboard-controls'
import { usePeriodQuery } from '@/app/components/admin/dashboard-controls/use-period-query'

export default function DashboardPage() {
  const { period, setPeriod } = usePeriodQuery()

  // 외곽 컨테이너 폭/패딩은 commonLayout 표준 (`/apps`, `/datasets` 페이지 패턴 차용)
  return (
    <div className="<commonLayout-page-container>">  {/* `/apps` 페이지에서 패턴 추출 */}
      <div className="flex items-center justify-end py-4">  {/* h1 없음, 우측 정렬만 */}
        <DashboardControls period={period} onPeriodChange={setPeriod} />
      </div>
      <KpiSection period={period} />
      <DeptObjectsChart period={period} />
      <ModelTokensChart period={period} />
      <DeptActivityTable period={period} />
    </div>
  )
}
```

> - h1 "대시보드" 제거 — 톱 네비 활성 메뉴에 이미 표시됨 (중복 회피)
> - 외곽 폭은 commonLayout 표준 컨테이너 따름 — 스튜디오/지식 페이지와 동일 좌우 여백/최대 폭 확보
> - `account-setting/index.tsx`에는 일절 손대지 않음 (모달 탭 패턴 폐기)

## API

대시보드 컨트롤은 백엔드 API 호출 없음. **URL 상태와 캐시 invalidation만 담당**.

## 컴포넌트 props

```typescript
// index.tsx
interface DashboardControlsProps {
  period: Period
  onPeriodChange: (period: Period) => void
}

// types.ts
export type TimePeriodKey = 'today' | 'last7days' | 'last30days' | 'last90days' | 'custom'

export interface Period {
  start: string       // "2026-04-23 00:00"
  end: string         // "2026-04-30 23:59"
  label: string       // "지난 7 일" — 자식 컴포넌트의 카드 기간 라벨 표시용
  key: TimePeriodKey
}
```

## 상태 관리 흐름 (URL query string)

```
[사용자가 드롭다운에서 "지난 30 일" 선택]
        ↓
DashboardControls의 onPeriodChange(period) 호출
        ↓
usePeriodQuery 내부에서 router.replace('?period=last30days')
        ↓
URL query 변경 → period prop 재계산
        ↓
KPI/차트/표가 받는 period 변경 → TanStack queryKey 변경 → 자동 refetch
        ↓
새 데이터 표시 (각 카드의 periodLabel도 "지난 30 일"로 갱신)
```

## use-period-query 훅 (URL state 기반)

`nuqs` 사용 가능. 직접 구현해도 가벼움.

```typescript
// use-period-query.ts (직접 구현 옵션 — nuqs 대안)
'use client'
import dayjs from 'dayjs'
import { useRouter, useSearchParams } from 'next/navigation'
import { useCallback, useMemo } from 'react'
import { DEFAULT_PERIOD_KEY, QUERY_DATE_FORMAT, TIME_PERIOD_OPTIONS } from './constants'
import type { Period, TimePeriodKey } from './types'

const MAX_DAYS = 90 // audit 보존 cap

function periodFromQuery(searchParams: URLSearchParams): Period {
  const key = (searchParams.get('period') ?? DEFAULT_PERIOD_KEY) as TimePeriodKey
  // custom인 경우: start/end 쿼리 파싱 + 90일 cap 검증 → 초과 시 last90days로 fallback
  // 그 외: TIME_PERIOD_OPTIONS에서 days 계산
  // ... (calcPeriod 함수와 동일 로직, key='custom' 분기 추가)
}

export function usePeriodQuery() {
  const router = useRouter()
  const searchParams = useSearchParams()

  const period = useMemo(() => periodFromQuery(searchParams), [searchParams])

  const setPeriod = useCallback((next: Period) => {
    const params = new URLSearchParams(searchParams)
    params.set('period', next.key)
    if (next.key === 'custom') {
      params.set('start', dayjs(next.start).format('YYYY-MM-DD'))
      params.set('end', dayjs(next.end).format('YYYY-MM-DD'))
    } else {
      params.delete('start')
      params.delete('end')
    }
    router.replace(`?${params.toString()}`, { scroll: false })
  }, [router, searchParams])

  return { period, setPeriod }
}
```

> `nuqs` 채택 시: `useQueryState('period', parseAsStringEnum([...]).withDefault('last7days'))` 형태로 더 간결. 프로젝트가 이미 `nuqs`를 도입했다면 그쪽 사용 권장 (`conventions.md § 모달 상태 → nuqs` 패턴과 일관).

## 새로고침 버튼

```tsx
'use client'
import { RiRefreshLine } from '@remixicon/react'
import { useIsFetching, useQueryClient } from '@tanstack/react-query'
import { memo, useCallback } from 'react'
import { cn } from '@/utils/classnames'

function RefreshButton() {
  const queryClient = useQueryClient()
  const isFetching = useIsFetching({ queryKey: ['dashboard'] })

  const handleClick = useCallback(() => {
    queryClient.invalidateQueries({ queryKey: ['dashboard'] })
  }, [queryClient])

  return (
    <button
      type="button"
      onClick={handleClick}
      className="flex h-9 w-9 items-center justify-center rounded-lg
                 hover:bg-state-base-hover-alt"
      aria-label="새로고침"
    >
      <RiRefreshLine className={cn('h-4 w-4 text-text-secondary', isFetching && 'animate-spin')} />
    </button>
  )
}

export default memo(RefreshButton)
```

> queryKey는 `['dashboard']` (2026-05-13 — `['admin', 'dashboard']`에서 admin 세그먼트 폐기).

## 기간 옵션 상수

```typescript
// constants.ts
export const QUERY_DATE_FORMAT = 'YYYY-MM-DD HH:mm'

export const TIME_PERIOD_OPTIONS = [
  { key: 'today', days: 0, label: '오늘' },
  { key: 'last7days', days: 7, label: '지난 7 일' },
  { key: 'last30days', days: 30, label: '지난 30 일' },
  { key: 'last90days', days: 90, label: '지난 90 일' },
  { key: 'custom', days: -1, label: '사용자 지정' },
] as const

export const DEFAULT_PERIOD_KEY: TimePeriodKey = 'last7days'
export const MAX_RANGE_DAYS = 90
```

## DatePicker

```tsx
import DatePicker from '@/app/components/base/date-and-time-picker'
```

- `custom` 선택 시에만 렌더
- 시작일/종료일 두 개 입력
- 범위 검증: `dayjs(end).diff(start, 'day') <= 90` 위반 시 입력 disabled + 토스트
- 선택 완료 시 트리거 라벨이 `"4월 1일 - 4월 30일"` 형식으로 변경

## 컨테이너 레이아웃

```tsx
// dashboard-controls/index.tsx
export default function DashboardControls({ period, onPeriodChange }: DashboardControlsProps) {
  return (
    <div className="flex items-center gap-2">
      <TimeRangePicker value={period.key} onChange={(key) => onPeriodChange(/* calc from key */)} />
      {period.key === 'custom' && (
        <DatePicker start={period.start} end={period.end} onChange={onPeriodChange} maxDays={90} />
      )}
      <RefreshButton />
    </div>
  )
}
```

> 본 컴포넌트는 자체 `<h1>` 없음. 페이지가 그림.

## 디자인 토큰

| 용도 | Tailwind |
|------|---------|
| 드롭다운 배경 | `bg-components-input-bg-normal` |
| 드롭다운 텍스트 | `text-components-input-text-filled` |
| 화살표 / 아이콘 | `text-text-quaternary` (활성 시 `text-text-secondary`) |
| 새로고침 hover | `bg-state-base-hover-alt` |

## 관련 노트

- [[4. 지식노트/Dify - 통계 기간 선택기 (TimeRangePicker).md]]
- [[4. 지식노트/Dify - 디자인 토큰과 차트·카드 색상 패턴.md]]
- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]]
