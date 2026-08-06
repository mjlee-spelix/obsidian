---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: KPI 드릴스루 (Phase 2 동적 인터랙션)
phase: 2
harness: [H-DASH-04, H-DASH-07, H-DASH-18, H-DASH-19, H-DASH-20]
date: 2026-05-04
last_updated: 2026-05-26
---
# KPI 드릴스루 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md|Design]]
> 순서: 인터랙션 골격(목업) → 드릴 차트 / 표(목업) → 백엔드 → 연동 → 테스트

## 1단계: 인터랙션 골격 (목업)

> 차트·표 데이터는 mock으로 두고, 클릭 / active / 토글 해제 / URL state 흐름 검증.

- [ ] 1. `useKpiDrillThrough` 훅 작성 — URL query string(`?metric`) 단일 진실, `toggle()`로 `router.replace`
- [ ] 2. ESC 핸들러 등록 — `e.defaultPrevented` 체크해서 다른 닫기 대상 우선 양보 후 metric 해제
- [ ] 3. `web/app/(commonLayout)/dashboard/page.tsx`에서 훅 호출 + `activeMetric` 전파
- [ ] 4. `KpiCardProps` 확장 — `isActive`, `metricKey`, `onActivate`, `aria-pressed`
- [ ] 5. `KpiCard` 시각 강조 — active 시 ring + 배경 변경, `cursor-pointer`, `role="button"`, `tabIndex={0}`
- [ ] 6. KPI 카드 4종에 `metricKey` 매핑 (`objects` / `users` / `calls` / `apps`)
- [ ] 7. KPI 카드 호버 툴팁 — active 시 "다시 클릭하여 종합 뷰로", 비active 시 메트릭명
- [ ] 8. `chart-area.tsx` — activeMetric 분기, 좌하 / 우하 / 하단 슬롯 교체 (`DRILL_SLOT_MAP`)
- [ ] 9. 활성 모드 빈 자리표시자 4종(좌하 / 우하 / 하단 표 / KPI 4 좌상 보조 카드) 렌더
- [ ] 10. 종합 뷰 복귀 3경로 검증 — (a) 재클릭 (b) 다른 KPI 클릭 (c) ESC
- [ ] 11. 전환 애니메이션 — 200ms fade (`key={activeMetric ?? 'overview'}`)
- [ ] 12. 키보드 접근성 — Tab 순회 + Enter/Space 토글 + ESC 해제
- [ ] 13. Deep link 검증 — `/dashboard?metric=calls`로 직접 진입 시 즉시 활성 모드 표시
- [ ] 14. 고정 높이 검증 — 종합 뷰 ↔ 활성 모드 ↔ 메트릭 4종 전환해도 차트 / 표 영역 높이 동일 (DRILL-LAYOUT-DRIFT 후보)

## 2단계: 드릴 컴포넌트 (목업)

> 차트 8종(좌+우, 메트릭별 2종) + 표 4종(하단). 좌하·우하 차트는 표시 전용 — 클릭 인터랙션 없음(H-DASH-20).
> 화면 설계 PDF v0.3 17p 영역 매핑을 따름.

### 2-A. objects (KPI 1)

- [ ] 15. `drill-charts/objects/dept-cumulative.tsx` — 부서별 누적 오브젝트 (가로 막대, owner_dept 기준)
- [ ] 16. `drill-charts/objects/top-owners.tsx` — Top 소유자 (가로 막대, 이름·부서·수치)
- [ ] 17. `drill-tables/dept-new-creations-table.tsx` — 부서 / 앱 / 지식 / 도구 / 신규 (i18n: `common.objectType.app/kb/tool`)
  - "신규"의 시점 정의는 PM 확인 항목 (ownership 등록 시점 vs Dify 생성 시점)
  - 부서명 가나다순 기본 정렬 (`localeCompare('ko-KR')`)

### 2-B. users (KPI 2 — 총 이용 앱 수)

- [ ] 18. `drill-charts/users/dept-adopted-apps.tsx` — 부서별 앱 이용 수 (가로 막대, `COUNT(DISTINCT app_id) WHERE actor_dept_id = dept`)
- [ ] 19. `drill-charts/users/top-users.tsx` — 앱 이용자 Top10 (가로 막대, Y축 이름만 + hover 3줄 이름/부서/값)
- [ ] 20. `drill-tables/dept-users-table.tsx` — 부서 / 부서원 / 부서원당 이용 앱 수 / Top 이용 앱 / 추세 (5컬럼)
  - 부서원당 이용 앱 수: `COUNT(DISTINCT target_app_id) / NULLIF(COUNT(DISTINCT actor_id), 0)`, 1자리 소수점, 분모 0 → "-"
  - Top 이용 앱: `DISTINCT ON (actor_dept_id) target_app_id ORDER BY COUNT(*) DESC` + `apps.name` JOIN
  - 추세: 이용 앱 수 전기간 대비 증감 %, KPI 카드 증감 뱃지 패턴 (H-DASH-07)
  - 부서명 가나다순 기본 정렬
  - 백엔드 단일 CTE 쿼리 — 5/19 부하 테스트 dept-user-activity 7쿼리 5.8초 병목 자연 해소 (NEW/CHURNED NOT IN 서브쿼리 제거)

### 2-C. calls (KPI 3 — 총 앱 호출량)

- [ ] 21. `drill-charts/calls/dept-call-count.tsx` — 부서별 앱 호출 수 (가로 막대, owner_dept 기준)
- [ ] 22. `drill-charts/calls/model-call-share.tsx` — 모델별 호출 분포 (가로 막대). `VAuditEnriched.model_id` 합산
- [ ] 23. `drill-tables/dept-call-rps-table.tsx` — 부서 / 호출 / RPS / Top 호출 앱 / 추세 (5컬럼)
  - RPS: `SUM(calls) / period_seconds`, 소수점 2자리. 분모 = `max((end-start)초, 1)`
  - Top 호출 앱: 부서 소유 앱 중 호출 1위. `ROW_NUMBER() OVER (PARTITION BY app_owner_dept_id ORDER BY SUM(calls) DESC)` + `apps.name` JOIN, 삭제 시 `"삭제된 앱"`
  - 추세: 호출 수의 전기간 대비 증감 % (H-DASH-07)
  - 부서명 가나다순 기본 정렬
  - 데이터 소스 = Layer 2 `spx_mv_kpi_calls_daily` (audit_events 직접 조회 금지)
  - api_call(nginx) 부서별 분류 가능성 PM 확인 항목 (현 마트는 콘솔 + end_user만 포함)

### 2-D. apps (KPI 4 — 인기 호출 앱)

- [ ] 24. KPI 4 카드 자체가 active로 강조 — 큰 자리에 앱명(truncate + hover 풀텍스트, `max-w-[55%]`), 뱃지 자리에 호출 수(회색 배지). 별도 보조 카드 신설 없음
- [ ] 25. `drill-charts/apps/app-call-top10.tsx` — 호출 수 Top10 (가로 막대)
- [ ] 26. `drill-charts/apps/top-error-apps.tsx` — 에러 발생 Top10 (가로 막대, ILIKE 6종 분류)
- [ ] 27. `drill-tables/app-stats-table.tsx` — 앱 / 부서 / 호출 / 이용자 / 에러 / 마지막 사용 (6컬럼)
  - 기본 정렬: 호출 내림차순 (서버 응답 순서 그대로). 앱 그레인이므로 부서명 가나다순 미적용
- [ ] 28. `app-stats-table` 행 클릭 → `router.push('/app/{appId}/overview')` (같은 탭), `role="link"` + `tabIndex={0}` + Enter 지원

### 2-E. 공통

- [ ] 29. 모든 표·차트 영역 고정 높이 + 내부 스크롤 (5/13 전역 정책)
- [ ] 30. 미배정 행 / 미배정 막대 처리 (H-DASH-04)
- [ ] 31. 증감률 / 추세 컬럼: 분모 0 시 "+신규" / "-" 분기 (H-DASH-07)

## 3단계: 백엔드

> audit_events 단일 SoT + RBAC JOIN 기본. 오브젝트만 `resource_ownership` 직접 조회.

- [ ] 32. Blueprint 등록 — `api/controllers/console/dashboard/drill.py`
- [ ] 33. 라우팅 결정 — 개별 11종 엔드포인트 vs 통합 엔드포인트 1종
- [ ] 34. `services/admin/drill/` 모듈 생성 (또는 `services/dashboard/drill/`로 통일 검토)
- [ ] 35. **objects 3종 서비스** — `dept-cumulative` / `top-owners` / `dept-new-creations` (resource_ownership 직접 조회)
- [ ] 36. **users 3종 서비스** — `dept-adopted-apps` / `top-users` / `dept-activity` (audit + RBAC JOIN)
  - `dept-activity`는 단일 CTE 쿼리로 구성 (design § users 표 CTE 본문 참조). 7쿼리 N+1 패턴 금지 (5/19 부하 테스트 5.8초 병목 원인)
- [ ] 37. **calls 3종 서비스** — `dept-call-count` / `model-call-share` / `dept-call-rps-table` (Layer 2 `spx_mv_kpi_calls_daily` enforce, model-call-share만 `VAuditEnriched` model_id)
- [ ] 38. **errors 2종 서비스 (KPI 4 dimension)** — `top-error-apps` / `app-stats` (audit + RBAC JOIN, ILIKE 6종)
- [ ] 39. 모든 audit 기반 쿼리에 디버깅 필터(`details->>'invokeFrom' != 'debugger'` + `details->>'triggeredFrom' != 'debugging'`) 적용. audit collector 미보강 시 OLTP 임시 대체(`invoke_from` / `triggered_from`)
- [ ] 40. 에러 분류 ILIKE 6종 — `audit_events.details->>'error'` 입력 (H-DASH-18). 6종: rate_limit / timeout / auth / quota / model_error / other
- [ ] 41. 미배정 부서 LEFT JOIN + fallback (H-DASH-04)
- [ ] 42. 외부 사용자 식별 — `actor_type = 'end_user'`
- [ ] 43. 증감률 분모 0 처리 (H-DASH-07)

### PM / audit 의존 항목 (보류)

- [ ] 43a. ⚠️ PM 확인 — `dept-new-creations` "신규"의 시점 정의 (ownership 등록 vs Dify 생성)
- [ ] 43b. ⚠️ PM 확인 — api_call(nginx) 부서별 분류 가능성. audit가 모든 호출을 캡처하지 못할 시 nginx 로그 보강 필요성
- [ ] 43c. ⚠️ PM 확인 — dataset_operator role 실제 사용 사용자 존재 여부 (UI 노출 부재)
- [ ] 43d. ⚠️ audit collector 보강 후 후속 — `SELECT DISTINCT details->>'appMode' FROM audit_events`로 H-CAND-audit-appmode-missing 검증
- [ ] 43e. ⚠️ audit collector 보강 후 후속 — `details->>'triggeredFrom'` 가용성 확인으로 H-CAND-audit-wf-debug-filter-missing 검증

## 4단계: 연동

- [ ] 44. 각 드릴 차트·표 컴포넌트의 mock → `useQuery` 훅 교체
- [ ] 45. queryKey 표준 검증 — `['dashboard', 'drill', metric, chart, params]`
- [ ] 46. 캐시 동작 검증 — 메트릭 전환 후 재진입 시 즉시 표시 (재요청 없음)
- [ ] 47. 페이지 헤더 새로고침 시 `['dashboard']` 접두사 전체 invalidate 검증
- [ ] 48. 페이지 헤더 기간 변경 시 모든 드릴 차트 refetch 검증
- [ ] 49. KPI 4 표 행 클릭 → `/app/{appId}/overview` 라우팅 검증 (같은 탭)
- [ ] 50. 미가용 메트릭 처리 — 데이터 없으면 "데이터 준비 중" 자리표시자 (PDF 18p 단서)

## 5단계: 테스트 (Harness 후보 검증 포함)

- [ ] 51. **DRILL-LAYOUT-DRIFT 검증** — 메트릭 전환 시 차트 / 표 영역 위치·높이·색상 토큰 동일성. PDF + 5/13 고정 높이 전역 정책 1:1 대조
- [ ] 52. 토글 해제 3경로 — 재클릭 / 다른 KPI / ESC
- [ ] 53. 키보드 접근성 — Tab + Enter/Space + ESC
- [ ] 54. Deep link — `/dashboard?metric=X` 직접 진입 / 새로고침 / 뒤로가기
- [ ] 55. KPI 4 표 행 클릭 (마우스 / Enter) → 모니터링 페이지 이동
- [ ] 56. 5분 캐시 일관성 — 헤더 새로고침이 드릴 차트도 invalidate
- [ ] 57. 미배정 부서 / 미배정 앱 행 노출 일관성 (H-DASH-04)

## 6단계: 검증된 결함 패턴만 defect-catalog 등록

- [ ] 58. 5단계에서 실제로 결함이 발생했거나 검증된 후보만 선별
- [ ] 59. 선별된 항목에 `H-DASH-XX` 정식 ID 부여 + `hdd/defect-catalog.md` 5필드 등록
- [ ] 60. spec 3파일의 frontmatter `harness: [...]`에 정식 ID 반영
- [ ] 61. `hdd/quality-criteria.md`에 KPI 드릴스루 체크리스트 추가

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md|Design]]
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]]
