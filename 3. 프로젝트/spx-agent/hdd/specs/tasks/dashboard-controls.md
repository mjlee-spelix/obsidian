---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: 대시보드 컨트롤
harness: []
date: 2026-04-30
last_updated: 2026-05-13
---
# 대시보드 컨트롤 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dashboard-controls.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/dashboard-controls.md|Design]]
> 순서: 컴포넌트 정리 → URL state 전환 → 페이지 헤더 통합 → 테스트 갱신

## 1단계: 컴포넌트 정리 (90일 cap에 맞춰 옵션 축소)

### 1-1. 상수 / 타입

- [ ] 1. `constants.ts` — `TIME_PERIOD_OPTIONS`를 5개로 축소
  - [ ] 1-1. `today` / `last7days` / `last30days` / `last90days` / `custom`만 유지
  - [ ] 1-2. `last4weeks`, `last3months`, `last12months`, `monthToDate`, `quarterToDate`, `yearToDate`, `allTime` 제거 (audit 90일 cap 초과)
  - [ ] 1-3. `MAX_RANGE_DAYS = 90` export
  - [ ] 1-4. `dayjs/plugin/quarterOfYear` import 제거 (더 이상 필요 없음)
- [ ] 2. `types.ts` — `TimePeriodKey` 유니온 타입 5개로 갱신, `Period.key` 필드 추가

### 1-2. URL query string 기반 상태 훅

- [ ] 3. `use-period-query.ts` — 신규 작성 (기존 `use-period.ts` 대체)
  - [ ] 3-1. `useSearchParams()` + `useRouter().replace()` 기반
  - [ ] 3-2. 쿼리 누락 시 `last7days`로 해석 (URL은 그대로 둠)
  - [ ] 3-3. `custom` 선택 시 `?period=custom&start=YYYY-MM-DD&end=YYYY-MM-DD`
  - [ ] 3-4. custom 범위 90일 초과 시 last90days로 fallback (방어)
  - [ ] 3-5. 또는 `nuqs` 채택 시 `useQueryState` 기반으로 간소화
- [ ] 4. 기존 `use-period.ts` (useState 기반) 제거

### 1-3. 컨트롤 컴포넌트

- [ ] 5. `time-range-picker.tsx` — 5개 옵션 렌더
  - [ ] 5-1. `SimpleSelect` 기반, `TIME_PERIOD_OPTIONS`에서 `{ value: key, name: label }` 생성
  - [ ] 5-2. 기본값: `last7days`
  - [ ] 5-3. `custom` 선택 시 트리거 라벨 `"4월 1일 - 4월 30일"`
- [ ] 6. `date-picker.tsx` — 시작/종료일 (custom일 때만 렌더)
  - [ ] 6-1. Dify `date-and-time-picker` 재사용
  - [ ] 6-2. 범위 90일 초과 시 disabled / 토스트 안내
- [ ] 7. `refresh-button.tsx` — 새로고침
  - [ ] 7-1. `queryClient.invalidateQueries({ queryKey: ['dashboard'] })` (admin 세그먼트 폐기)
  - [ ] 7-2. `useIsFetching({ queryKey: ['dashboard'] })` 기반 회전 애니메이션
- [ ] 8. `index.tsx` — 컨트롤 묶음
  - [ ] 8-1. `flex items-center gap-2` 레이아웃 (자체 `<h1>` 없음)
  - [ ] 8-2. props: `period`, `onPeriodChange`
  - [ ] 8-3. `period.key === 'custom'`일 때만 DatePicker 렌더

## 2단계: 페이지 헤더 슬롯 통합

- [ ] 9. `app/(commonLayout)/dashboard/page.tsx`에 페이지 헤더 슬롯 작성
  - [ ] 9-1. `<header>` 좌측 `<h1>대시보드</h1>` + 우측 `<DashboardControls />`
  - [ ] 9-2. `usePeriodQuery()`를 페이지 컴포넌트가 호출 (단일 진실)
  - [ ] 9-3. `period` prop을 `<KpiSection />`, 차트, 표 모두에 전달
- [ ] 10. 설정 모달(`account-setting/index.tsx`) 관련 변경은 본 컴포넌트 범위 밖 (모달 DASHBOARD 탭 제거는 별건 작업)

## 3단계: 테스트 갱신 (기존 6/6 통과 → 신규 정책 반영)

- [ ] 11. `__tests__/index.spec.tsx` 갱신
  - [ ] 11-1. "should render all 9 period options" → "should render all 5 period options"로 변경 (today / last7days / last30days / last90days / custom)
  - [ ] 11-2. `'option-last3months'` 클릭 테스트 → `'option-last30days'` 또는 `'option-last90days'`로 교체
  - [ ] 11-3. RefreshButton invalidate spy: `{ queryKey: ['dashboard'] }` 확인 (이미 그렇게 호출하지만 테스트 케이스 이름 `'admin dashboard key'` → `'dashboard key'`로 수정)
  - [ ] 11-4. `usePeriod` 훅 테스트 → `usePeriodQuery` 훅 테스트로 교체. `useSearchParams` / `useRouter` mock 필요
  - [ ] 11-5. custom 선택 시 start/end 쿼리 추가 케이스 / 90일 초과 fallback 케이스 추가
- [ ] 12. `pnpm vitest run app/components/admin/dashboard-controls` 통과 확인

## 4단계: 검증

- [ ] 13. 화면 확인 — `/dashboard` 페이지 헤더에 "대시보드" + 우측 컨트롤 1줄 배치
- [ ] 14. 드롭다운 5개 옵션 노출 (90일 초과 옵션 없음)
- [ ] 15. `last30days` 선택 시 URL이 `/dashboard?period=last30days`로 변경
- [ ] 16. 브라우저 새로고침 후에도 선택 유지 (URL state)
- [ ] 17. 브라우저 뒤로가기 동작 자연스러움
- [ ] 18. `custom` 선택 + 90일 범위 입력 시 자식 컴포넌트 refetch
- [ ] 19. `custom` 선택 + 90일 초과 입력 시 disabled 또는 fallback
- [ ] 20. 새로고침 버튼 → 모든 대시보드 쿼리 일괄 invalidate

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/dashboard-controls.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/dashboard-controls.md|Design]]
- [[4. 지식노트/Dify - 통계 기간 선택기 (TimeRangePicker).md]]
