---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: 대시보드 컨트롤 (기간 선택 + 새로고침)
harness: []
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_image: "images/Dify/(Dify) 모니터링 - 챗봇.png"
mount_environment: "/dashboard 톱레벨 라우트 페이지 헤더 슬롯"
date: 2026-04-30
last_updated: 2026-05-22
---
# 대시보드 컨트롤 — Requirements

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/dashboard-controls.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dashboard-controls.md|Tasks]]

## 배경 및 책임 범위

- 대시보드의 모든 컴포넌트(KPI / 부서별 오브젝트 / 모델별 토큰 / 부서별 활동)가 공통으로 받는 **기간 필터 + 새로고침** 컨트롤만 담당.
- 페이지 자체 `<h1>` 제목은 `app/(commonLayout)/dashboard/page.tsx`가 그림. 본 컴포넌트는 **페이지 헤더 슬롯에 들어가는 컨트롤 묶음(toolbar)**.
- 마운트 환경이 라우트 페이지이므로 URL query string으로 상태 관리 (Phase 2 drill-through deep link와 호환).

## 마운트 환경

- **위치**: `/dashboard` 페이지 헤더 슬롯 (페이지 우상단)
- 페이지 컴포넌트(`app/(commonLayout)/dashboard/page.tsx`)가 좌측에 **페이지 소개 문구**, 우측에 `<DashboardControls />`를 한 줄에 배치 (`sticky top-0 z-10 flex items-center justify-between bg-background-body px-12 pt-7 pb-5`)
- 비로그인 가드는 상위 `(commonLayout)`의 `AppInitializer`가 자동 처리 (`/signin` 리다이렉트)

```
┌──── /dashboard 페이지 헤더 ─────────────────────────────┐
│ 앱, 지식, 도구의 사용 현황과 …  [지난 7 일 ▼] [📅] [🔄]   │
└── page.tsx 좌측 문구 ── DashboardControls (헤더 우측) ────┘
┌──── 콘텐츠 영역 ─────────────────────────────────────────┐
│ ┌─KPI─┐ ┌─KPI─┐ ┌─KPI─┐ ┌─KPI─┐                        │
│ └─────┘ └─────┘ └─────┘ └─────┘                        │
└──────────────────────────────────────────────────────────┘
```

## 페이지 소개 문구

- **위치**: `/dashboard` 페이지 헤더 좌측 (DashboardControls의 sibling, 같은 sticky 행)
- **렌더링**: `page.tsx`의 `<div className="grow truncate text-text-primary system-xl-semibold">{t('common.menus.dashboardDescription')}</div>`
- **i18n key**: `common.menus.dashboardDescription`
  - 한국어: `앱, 지식, 도구의 사용 현황과 부서별 활동을 한눈에 확인합니다.`
  - 영어: `Monitor app, knowledge, and tool usage with department activity at a glance.`
- 별도 `<h1>` 헤더 미사용 — 본 문구가 페이지 진입 컨텍스트를 대신함

## 화면 요구사항

### 구성 요소 (좌→우)

1. **기간 선택 드롭다운** — 5개 옵션 + 사용자 지정
2. **사용자 지정 DatePicker** — 시작일/종료일 별도 선택 (custom 선택 시에만 활성)
3. **새로고침 버튼** — 대시보드 전체 쿼리 invalidate

## 기간 옵션

> audit_events 자연 보존 정책이 **90일**이라 가장 긴 옵션도 90일로 잘림. UX 단순화를 위해 90일을 초과할 수 있는 옵션(`last3months`, `last12months`, `monthToDate`, `quarterToDate`, `yearToDate`, `allTime`)은 **전부 제거**.

| # | key | 한국어 라벨 | 기간 |
|---|-----|------------|-----|
| 1 | `today` | 오늘 | 0일 (오늘 00:00~23:59) |
| 2 | `last7days` | 지난 7 일 | 7일 |
| 3 | `last30days` | 지난 30 일 | 30일 |
| 4 | `last90days` | 지난 90 일 | 90일 |
| 5 | `custom` | 사용자 지정 | DatePicker로 임의 범위 (90일 이내) |

- **기본값**: `last7days` (지난 7 일)
- `custom` 선택 시 트리거 라벨이 `"4월 1일 - 4월 30일"` 형식으로 변경
- 24h 옵션은 `today`로 처리 (자정 기준 당일)

### 기간 max 정책 (audit 90일 cap)

- **모든 차트가 동일 cap (90일) 적용** — 마트 입력이 audit_events 단일 SoT로 통일됐기 때문 (`design.md § 4.6` 참조)
- 본 컴포넌트는 90일 초과 옵션을 **노출하지 않음** → 사용자가 cap을 만질 수 없음
- `custom` 선택 시 DatePicker 자체에서 90일 초과 범위 입력 시 disabled / 토스트 안내 (구현 시 확정)
- ⚠️ **PM 확인 후보**: custom 범위 90일 초과 시 UX 디테일 (잠금 vs 자동 클램프 vs 토스트) — 우선 disabled로 진행하고 추후 확인

## 상태 관리 (URL query string)

라우트 페이지 환경이므로 URL query string을 단일 진실로 사용. Context/prop drilling 금지.

- **쿼리 키**: `?period=last7days` 또는 `?period=custom&start=2026-04-01&end=2026-04-30`
- **구현 옵션**: `nuqs` 패키지 또는 직접 작성한 `usePeriodQuery` 커스텀 훅 (둘 다 `useSearchParams` + `router.replace` 기반)
- **기본값**: 쿼리 누락 시 `last7days`로 해석 (URL은 유지)
- **이점**: Phase 2 drill-through deep link / 새로고침 시 상태 보존 / 브라우저 뒤로가기 동작 자연스러움

```typescript
// /dashboard?period=last7days
// /dashboard?period=last30days
// /dashboard?period=custom&start=2026-04-01&end=2026-04-30

DashboardPage
  ↓ const { period, setPeriod } = usePeriodQuery()  // URL query 단일 진실
  ├─ <DashboardControls period={period} onPeriodChange={setPeriod} />  (페이지 헤더 슬롯)
  ├─ <KpiSection period={period} />
  ├─ <DeptObjectsChart period={period} />
  ├─ <ModelTokensChart period={period} />
  └─ <DeptActivityTable period={period} />
```

## 새로고침 버튼

- 클릭 시 `queryClient.invalidateQueries({ queryKey: ['dashboard'] })` — admin 세그먼트 폐기 (2026-05-13)
- 대시보드 컴포넌트 전체 동시 refetch
- 아이콘: `RiRefreshLine`
- 진행 중 표시: `useIsFetching({ queryKey: ['dashboard'] })` 시 아이콘 회전

## 비기능 요구

- 다크/라이트 모드 자동 대응 (디자인 토큰만 사용, raw 색상값 금지)
- 모바일 대응: 좁은 화면에서 자연스러운 줄바꿈
- 캐시 정책: 자식 컴포넌트의 `staleTime: 5 * 60 * 1000` 유지

## 방어할 Harness

| ID | 결함 | 이 컴포넌트에서의 방어 |
|----|------|---------------|
| (해당 없음) | — | 컨트롤만 담당, 데이터 집계 없음. 90일 cap은 옵션 노출 차단으로 사전 방어 |

## 관련 노트

- [[4. 지식노트/Dify - 통계 기간 선택기 (TimeRangePicker).md]]
- [[4. 지식노트/Dify - 디자인 토큰과 차트·카드 색상 패턴.md]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|KPI 카드 requirements]] (period prop 수신처)
