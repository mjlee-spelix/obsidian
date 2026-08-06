---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
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

# 차트 드로어 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/chart-drawer.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/chart-drawer.md|Design]]
> 순서: 드로어 골격(목업) → 컴포넌트 디테일 → 트리거 차트 연결 → 백엔드 → 연동 → 테스트

## 1단계: 드로어 골격 (목업)

> 컨텍스트 state, 슬라이드 인/아웃, 닫기 메커니즘이 작동하는지 먼저 검증. 데이터는 mock.

- [ ] 1. `chart-drawer/types.ts` — `DrawerContext`, `DrawerMetric`, `DrawerData`, `LogEntry` 타입 정의
- [ ] 2. `use-chart-drawer.ts` 훅 — `context` state, `open`/`close`/`toggle`. **ESC 핸들러 미설치**
- [ ] 3. `useEffect`로 메트릭/기간 변경 시 자동 닫기
- [ ] 4. `chart-drawer/index.tsx` 컨테이너 — Portal + 백드롭 + 슬라이드 인 패널
- [ ] 5. 슬라이드 인 애니메이션 — 200~300ms `transition-transform duration-250` + `translate-x-0`/`translate-x-full`
- [ ] 6. 백드롭 클릭으로 닫기 + 백드롭 페이드
- [ ] 7. z-index 결정 — 모달(`z-1002`) 위에 `z-1003` (디자인 토큰 검토)
- [ ] 8. 빈 골격에서 mock 컨텍스트로 열고 닫기 시각 검증
- [ ] 8-1. **배경 차트 흐림 처리** — 드로어 열림 시 dept-activity 테이블 등 배경 차트에 `opacity-40` 또는 `blur-sm` + `pointer-events-none` 적용. 닫힘 시 정상 복귀
- [ ] 8-2. **화면 폭 확인** — 드로어 480px가 차트 영역 옆에 보이는 폭인지 실 화면 측정. 모달 폭이 부족하면 드로어가 차트 완전히 덮어 흐림 불필요 — 결정 후 spec 갱신
- [ ] 8-3. **사용자 기간 일관성 확인** — 부서별 호출 수 등 모든 차트가 사용자 기간 따르는지 확인 (정책 미적용 차트 있으면 수정 필요)

## 2단계: 드로어 내부 컴포넌트 (목업)

- [ ] 9. `drawer-header.tsx` — 부서명(+`팀` 접미사) + 메트릭 칩 + CSV 버튼 + 닫기 버튼 (`RiCloseLine`)
  - 부서명 형식: `${department_name}팀 활동`. 미배정은 "미배정 활동" ("팀" 접미사 없음)
  - CSV 버튼은 헤더 우측 (닫기 버튼 옆) 아이콘 버튼
- [ ] 10. 닫기 버튼에 `closeButtonRef` 연결 → 드로어 열릴 때 자동 포커스
- [ ] 11. `context-chips.tsx` — **칩 4종**: 부서 / 지표 / 기간 / 에러 (표시 전용, 칩 클릭 비활성)
- [ ] 12. `summary-cards.tsx` — 호출/사용자 수/에러 3종 카드 (KPI 카드 디자인 토큰 재사용)
  - **증감률 포함**: 각 카드에 이전 기간 대비 % 표시 (예: +12.6%)
  - 색상: 에러 카드는 양수 빨강(증가=나쁨), 호출/사용자는 양수 녹색
- [ ] 13. 에러 카드 — **실패 호출 수 절대값** (에러율 % 아님). `error_count` 표시 + 증감률
- [ ] 14. 증감률 N/A 처리 — 이전 기간 데이터 없거나 분모 0이면 "—" 표시 (H-DASH-05/H-DASH-07)
- [ ] 15. `log-list.tsx` — 6~10건 로그 표시 (시각 / 사용자 / 앱·메트릭 / 요약)
- [ ] 16. 로그 항목에 `has_error` 시 시각 표시 (예: 좌측 빨간 도트)
- [ ] 17. 검색 입력 — 클라이언트 필터링 (요약 텍스트 부분 일치)
- [ ] 18. 빈 상태 — "로그 없음" 안내 카드
- [ ] 19. `more-button.tsx` "더 보기 →" — 클릭 시 placeholder 토스트 ("감사 로그 풀뷰는 추후 지원 예정")
  - 클릭 시 컨텍스트(부서/메트릭/기간/에러 필터) 그대로 풀뷰 페이지로 전달 (구현은 후속)
- [ ] 20. `csv-button.tsx` — 헤더 내부에서 사용. 클릭 시 안내 토스트 ("CSV 다운로드는 추후 지원 예정")

> ❌ "전체 보기" 별도 버튼 없음 — "더 보기"가 곧 풀뷰 페이지네이션 진입점

## 3단계: 트리거 차트 연결

- [ ] 20. `dept-objects-chart` 컴포넌트에 `onBarClick(department_id, department_name)` prop 추가 (Phase 1 spec 갱신 — 별도 작업)
- [ ] 21. ECharts `click` 이벤트 핸들러 연결 — 막대 컴포넌트에서만 발동, 빈 영역 무시
- [ ] 22. drill-through 활성 모드의 좌 차트(부서별 ___)들에도 동일 `onBarClick` 추가 (4개 컴포넌트)
- [ ] 23. `DashboardPage`에서 `useChartDrawer(activeMetric, period)` 호출 + 트리거 핸들러 작성
- [ ] 24. 트리거 핸들러: `metric = activeMetric ?? 'objects-overview'` + `drawer.toggle({ department_id, department_name, metric })`
- [ ] 25. "미배정" 막대 처리 — `department_id: null` + `department_name: '미배정'` 컨텍스트로 전달
- [ ] 26. 시각 검증 — 종합 뷰 + 4개 활성 모드 모두에서 막대 클릭 시 드로어 열림

## 4단계: 접근성

- [ ] 27. `FocusTrap` 도입 — `@base-ui/react` 또는 `react-focus-lock` 중 코드베이스 패턴 따름
- [ ] 28. `role="dialog"` + `aria-modal="true"` + `aria-labelledby="chart-drawer-title"` 적용
- [ ] 29. 드로어 열릴 때 닫기 버튼으로 포커스 자동 이동 (1단계 항목 10 완성형)
- [ ] 30. 드로어 닫힐 때 포커스를 트리거 막대로 복귀 (가능한 경우)
- [ ] 31. Tab 순회 검증 — 닫기 → 검색 → 로그 → 전체 보기 → CSV → 닫기 (순환)

## 5단계: 백엔드

> 승랑님 스키마 의존. 받기 전엔 mock 응답 유지.

- [ ] 32. `controllers/console/dashboard/drawer.py`에 `/drawer` 라우트 등록 — `department_id`, `metric`, `start`, `end` 쿼리
- [ ] 33. `services/admin/drawer/` 모듈 생성
- [ ] 34. **요약 3종 메서드** — KPI 카드 서비스 메서드 재사용 + `WHERE department_id = :department_id` 추가, **증감률 포함**
  - [ ] 34-1. `get_dept_calls()` — `get_api_calls()` 부서 필터 + 이전 기간 비교로 `calls_diff_percent` 계산
  - [ ] 34-2. `get_dept_users()` — `get_users()` 부서 필터 + 이전 기간 비교로 `users_diff_percent` 계산
  - [ ] 34-3. `get_dept_error_count()` — **실패 호출 절대값** (에러율 % 아님). `messages.status='error'` + `workflow_runs.status='failed'` 카운트 + 이전 기간 비교로 `errors_diff_percent` 계산
  - [ ] 34-4. 증감률 분모 0 보호 — `None` 반환 (H-DASH-05/H-DASH-07)
  - [ ] 34-5. 모든 카드 사용자 period 따름
- [ ] 35. **로그 6~10건 메서드** — messages + workflow_runs UNION ALL → 시각 정렬 → LIMIT 10
- [ ] 36. 미배정 fallback — `department_id IS NULL` 시 `resource_ownership.id IS NULL` 행 (H-DASH-04)
- [ ] 37. 디버깅 필터 적용 — `invoke_from != 'debugger'` + `triggered_from = 'app-run'` (H-DASH-03)
- [ ] 38. ADVANCED_CHAT 분기 — `messages`만 카운트 (H-DASH-01)
- [ ] 39. 응답 직렬화 (Pydantic `DrawerData`) — `context` 에코 포함

> CSV 다운로드는 백엔드 미구현 (SESSION_HISTORY 핵심 결정).
> "전체 보기" 풀뷰 백엔드도 미구현 (별도 spec 후속).

## 6단계: 연동

- [ ] 40. `use-drawer-data.ts` 훅 — TanStack Query, `enabled: !!context`, queryKey 표준
- [ ] 41. queryKey 검증 — `['dashboard', 'drawer', department_id, metric, period]`
- [ ] 42. mock → 실제 API 교체
- [ ] 43. 같은 부서 재진입 시 캐시 사용 검증 (5분 staleTime)
- [ ] 44. 빠른 연속 클릭 시 응답 순서 검증 (DRAWER-CONTEXT-MISMATCH 후보)
- [ ] 45. 5분 캐시 일관성 — 헤더 새로고침이 드로어도 invalidate

## 7단계: 테스트 (Harness 후보 검증 포함)

> requirements의 "Harness 후보" 참조. 검증 통과한 항목만 8단계에서 정식 ID 부여.

- [ ] 46. **DRAWER-CONTEXT-MISMATCH 검증**: 막대 빠른 연속 클릭 → 응답 도착 순서/취소 정책 확인. 로딩 스켈레톤 노출로 잔여 데이터 가림 여부.
- [ ] 47. **DRAWER-FOCUS-TRAP 검증**: Tab이 드로어 외부로 빠지지 않는지 키보드 접근성 테스트
- [ ] 48. **DRAWER-CSV-EXPECTATION 검증**: CSV 버튼 UX 적절성 (토스트 vs disabled + 툴팁) 시각 검증
- [ ] 49. ESC 키 동작 확인: 드로어가 ESC 가로채지 않고 모달 닫기 동작 정상 유지 (회피 결정 검증)
- [ ] 50. 메트릭/기간 변경 시 자동 닫기 동작 확인
- [ ] 51. 같은 막대 재클릭 → 닫기, 다른 막대 클릭 → 컨텍스트 전환 (닫지 않음) 검증
- [ ] 52. "미배정" 막대 클릭 → `department_id: null` 컨텍스트 + 응답 데이터 정상
- [ ] 53. **증감률 표시 검증** — 양수/음수/N/A 3가지 상태 시각 확인. 에러는 양수=빨강, 호출/사용자는 양수=녹색
- [ ] 54. **배경 차트 흐림 검증** — 드로어 열림/닫힘 토글 시 dept-activity 등 배경 차트 opacity 정상 전환

## 8단계: 검증된 결함 패턴만 defect-catalog 등록

> ❗ **검증 안 된 가설은 등록하지 않음**. catalog 신뢰도 보호.

- [ ] 53. 7단계에서 실제로 결함이 발생했거나 소스 분석으로 위험이 입증된 후보만 선별
- [ ] 54. 선별된 항목에 `H-DASH-XX` 정식 ID 부여 + `hdd/defect-catalog.md`에 5필드 등록
- [ ] 55. spec 3파일의 frontmatter `harness: [...]`에 정식 ID 반영 (현재는 `[]`)
- [ ] 56. `hdd/quality-criteria.md`에 차트 드로어 컴포넌트 체크리스트 추가

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/chart-drawer.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/chart-drawer.md|Design]]
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]]
