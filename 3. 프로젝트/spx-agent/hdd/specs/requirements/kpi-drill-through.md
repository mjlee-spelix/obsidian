---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: KPI 드릴스루 (Phase 2 동적 인터랙션)
phase: 2
harness: [H-DASH-04, H-DASH-07, H-DASH-18, H-DASH-19, H-DASH-20]
harness_candidates: [DRILL-LAYOUT-DRIFT, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_pdf: "0. Inbox/화면설계_v0.3.pdf"
reference_pdf_pages: "17"
date: 2026-05-04
last_updated: 2026-05-22
---
# KPI 드릴스루 — Requirements

> Phase 2 동적 인터랙션 #1 — KPI 카드 클릭 시 차트·테이블 영역의 메트릭이 해당 KPI에 맞게 전환됨.
>
> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/kpi-drill-through.md|Tasks]]
> 참조: PDF v0.3 17p

## 마운트 환경

- 페이지: `/dashboard` 톱레벨 라우트 (`web/app/(commonLayout)/dashboard/page.tsx`)
- 접근 권한: 로그인 사용자 전체 (`AppInitializer` 가드만 적용, 별도 RoleRouteGuard 없음)
- 상태 관리: `activeMetric`을 URL query string으로 (`?metric=calls`) 단일 진실. React state는 query string에서 파생
- 페이지 헤더의 새로고침 / 기간 선택기와 자연스럽게 연동
- ESC 키 자체 핸들링 가능 (모달 환경 아님)

## 화면 요구사항

### 진입 / 종합 뷰 (기본 상태)

기본 상태는 **종합 뷰** — 어떤 KPI도 선택되지 않은 상태(`?metric` 미존재). 차트·테이블은 정적 대시보드와 동일하게 표시:

| 영역 | 컴포넌트 |
|------|---------|
| 좌하 차트 | 부서별 오브젝트 분포 (스택 막대, 클릭 인터랙션 없음) |
| 우하 차트 | 모델별 토큰 사용량 |
| 하단 표 | 부서별 활동 |

### 활성 모드 (KPI 선택됨)

KPI 카드 4종 중 하나를 클릭하면 해당 KPI의 메트릭 모드로 진입. URL = `?metric={objects|users|calls|apps}`.

- 클릭한 KPI 카드 = `active` 상태 (시각적 강조)
- 차트·테이블의 위치/색상 체계는 유지 (메트릭만 교체)
- 좌하·우하 차트 자체에는 클릭 인터랙션 없음 (chart-drawer 보류 — H-DASH-20)
- 표 행 클릭은 KPI 4(인기 호출 앱)에서만 활성화 — 같은 탭에서 `/app/{appId}/overview`로 이동. KPI 1·2·3 표 행 클릭은 동작 없음

### 메트릭 카탈로그

| 활성 KPI (`metric`) | 좌하 차트 | 우하 차트 | 하단 표 |
|---|---|---|---|
| 총 오브젝트 (`objects`) | 부서별 누적 오브젝트 (가로 막대) | Top 소유자 (가로 막대) | 부서별 신규 생성 (부서 / 앱 / 지식 / 도구 / 신규) |
| 총 이용 앱 수 (`users`) | 부서별 앱 이용 수 (가로 막대) | 앱 이용자 Top10 (가로 막대) | 부서별 앱 이용 현황 (부서 / 부서원 / 부서원당 이용 앱 수 / Top 이용 앱 / 추세) |
| 총 앱 호출량 (`calls`) | 부서별 앱 호출 수 (가로 막대) | 모델별 호출 분포 (가로 막대) | 부서별 앱 호출 현황 (부서 / 호출 / RPS / Top 호출 앱 / 추세) |
| 인기 호출 앱 (`apps`) | 호출 수 Top10 (가로 막대) | 에러 발생 Top10 (가로 막대) | 앱별 이용 현황 (앱 / 부서 / 호출 / 이용자 / 에러 / 마지막 사용) |

> KPI 4 활성 시 좌상의 KPI 4 카드 자체가 active로 강조 — 큰 자리에 앱명(truncate), 뱃지 자리에 호출 수(회색 배지). 별도 보조 카드 없이 KPI 카드 그리드 그대로 유지.

### 종합 뷰 복귀

세 가지 경로 모두 지원:

1. **재클릭** — active KPI 카드를 다시 클릭하면 토글 해제
2. **다른 KPI 카드 클릭** — 별도 해제 없이 새 메트릭으로 자연 전환
3. **ESC 키** — `?metric` 파라미터 제거 후 종합 뷰로 복귀 (모달 마운트 폐기로 ESC 자유 사용)

> ESC 적용 우선순위: 페이지 안에 다른 닫기 대상(예: 향후 추가될 드로어)이 열려 있으면 그쪽이 먼저 닫히고, 없을 때만 drill-through 해제. 자세히는 design 참조.

### 시각 강조

- **active 카드**: `border-transparent ring-inset ring-2 ring-components-button-primary-bg-hover shadow-lg` — 배경 토큰 변경 없음(`bg-components-card-bg` 유지). `ring-inset`으로 카드 안쪽 ring → 부모 컨테이너 padding 무관 → 잘림 0 + 레이아웃 변동 0
- **비active 카드**: 호버 시 클릭 가능 표시 (`cursor-pointer hover:shadow-lg`)
- 차트 전환 애니메이션 200ms 페이드 (`key={activeMetric}`로 컨테이너 리마운트)

### 고정 높이 정책 (5/13 전역)

- 카드 / 차트 / 표 모두 **고정 높이**. 데이터 양에 따라 영역 크기 변동 금지.
- 콘텐츠 초과 시 영역 내부 스크롤. 종합 뷰 ↔ 활성 모드 전환해도 컨테이너 높이 유지.
- 메트릭별 표 컬럼 수가 달라도 표 영역 높이 동일 (5/13 글로벌 레이아웃 정책).
- **행 수 고정 (5/20 UI 정합)**: 부서별 오브젝트 차트 **6행** / 모델별 토큰 차트 **6행** / 부서별 활동 표 **10행** + 나머지는 내부 스크롤. 카드 크기는 데이터 양과 무관하게 고정.
- **빈 상태 골격 (2026-06-09)**: 드릴 차트/표는 데이터 없어도 **제목 항상 유지**(`if (length===0) return <맨 박스>`로 제목까지 날리는 early-return 금지). **부서 그레인**(dept-call-count, dept-cumulative, dept-adopted-apps, dept-* 표) = 활성 부서 전수(0막대/`-`). **랭킹·엔티티**(Top10, 앱별, 모델별) = 활성만 + 기간 명시 안내 문구. 상세·판단 프레임워크 → `conventions.md § 빈 상태 / 골격 / 결측치 정책` (SoT).

## 정렬 규칙

- **부서 그레인 표 4종 중 부서 그레인은 부서명 가나다순(`localeCompare('ko-KR')`) 기본 적용** — `dept-users-table`, `dept-call-rps-table`, `dept-new-creations-table` (5/22 코드 정합)
- **앱 그레인 표(`app-stats-table`)** 는 서버 응답 순서 그대로(호출 수 내림차순) — 가나다순 비적용
- 차트 데이터: 주 지표 기준 내림차순 유지 (호출 수 / 이용자 수 / 에러율 등). 단 좌하 부서 가로 막대는 부서 가나다순 적용 후 막대 길이로 시각화
- 동률 시 2차 정렬 = 이름 오름차순 (한국어 ㄱ→ㅎ)
- 미배정/외부 sentinel 행은 항상 마지막

## i18n 라벨 정합

- `App/KB/Tool` 라벨은 i18n key로 관리 (하드코딩 금지)
- key: `common.objectType.app` / `common.objectType.kb` / `common.objectType.tool`
- 한국어: **앱** / **지식** / **도구**
- 영어: **App** / **KB** / **Tool**
- 적용 위치: 부서별 오브젝트 차트의 카드 헤더 React 범례 / 부서별 신규 생성 표 컬럼 / 기타 전수

## 차트 UI 정합 (Y축 라벨 + hover 툴팁)

### Top10 우하 차트 (이용자/소유자 — 사람 단위)

- **Y축 라벨 = 이름만** (부서명/소속 같은 부가 정보 제외)
- **hover 툴팁 3줄**: 이름 / 부서 / 수치
  - 적용 차트: `top-users` (`drill-charts/users/top-users.tsx`) — `이용 앱: N개`
  - 향후 `top-owners` 동일 패턴 적용 예정 (현재 spec 명시, 코드 확인 후)

### 좌하 부서별 막대 차트

- Y축 라벨 = 부서명 (부서 가나다순)
- hover 툴팁 2줄: 부서 / 수치 (`drill-charts/users/dept-adopted-apps.tsx` 현재 구현)
- 부가 정보가 차트에 시각적으로 이미 명시되어 있으므로 2줄 유지

## 키보드 접근성

| 키 | 동작 |
|----|------|
| `Tab` | KPI 카드 4종 순회 |
| `Enter` / `Space` | 포커스된 KPI 카드 활성화/해제 (토글) |
| `Esc` | active 메트릭 해제 → 종합 뷰 복귀 (다른 닫기 대상 우선) |

- KPI 카드는 `role="button"` + `tabIndex={0}` + `aria-pressed={isActive}`
- 키보드만으로 active 해제 가능: 같은 카드에 포커스 후 Enter/Space 재누름 또는 ESC

## 데이터 소스 규칙

### 마트 입력 단일 SoT

- **audit_events 단일 SoT + RBAC JOIN**이 기본 (사용자/호출/에러 메트릭). 백엔드 서비스는 audit 마트(또는 OLTP 임시 대체)에서 조회.
- 오브젝트 메트릭(KPI 1)만 `resource_ownership` 직접 조회 — 오브젝트는 state라 audit 본질 불가.
- 부서 매핑은 `department_members` JOIN으로. 미배정 부서는 LEFT JOIN + "미배정" fallback (H-DASH-04).

### 디버깅 필터 (H-DASH-03)

- audit 마트: `details->>'invokeFrom' != 'debugger'` + `details->>'triggeredFrom' != 'debugging'` (collector 보강 후)
- OLTP 임시 대체 시: `invoke_from != 'debugger'` + `triggered_from = 'app-run'`
- audit collector 미보강 상태에서는 후보 Harness `H-CAND-audit-appmode-missing`, `H-CAND-audit-wf-debug-filter-missing` 적용

### 외부 사용자 처리

- 외부 end_user는 audit `actor_type = 'end_user'`로 식별 (마트 톱레벨 컬럼)
- 부서별 차트에서는 외부 사용자 활동을 별도 행으로 노출하지 않고, 사용자 수 합계에는 포함 (필요 시 디자인 단계에서 별도 표시 결정)

### 에러 분류 (H-DASH-18, ILIKE 6종)

- 입력: `audit_events.details->>'error'`(향상 collector 보강 후)
- Dify가 예외 타입을 DB에 보존하지 않으므로 raw 텍스트 기반 ILIKE 분류 6종으로 정규화: rate_limit / timeout / auth / quota / model_error / other
- 표·차트의 "에러" / "에러율" 컬럼은 모두 같은 분류 규칙 사용

### 기간 정책

- 모든 차트·표는 사용자 `period`(`dashboard-controls`에서 받은 `start`/`end`)를 따름. 24h 고정 차트 없음.
- 기간(`period`)과 메트릭(`metric`)은 독립 — 메트릭 전환해도 기간 유지.
- 증감률 계산 시 분모 0 → "+신규" / "-" 분기 (H-DASH-07)

### 미배정 처리 (H-DASH-04)

- 모든 부서별 메트릭에서 "미배정" 행/막대 유지
- 색상 체계 통일 (App=blue / KB=teal / Tool=orange) — 메트릭과 무관하게 일관 토큰

## 비즈니스 규칙

### 메트릭 전환 시 데이터 일관성

- 종합 뷰 ↔ 활성 모드 전환 시 차트의 데이터 fetching은 별도 (queryKey가 metric 포함)
- 종합 뷰의 차트 데이터는 캐시 유지 → 활성 모드에서 종합 뷰 복귀 시 재요청 없이 표시
- 페이지 헤더의 새로고침은 `['dashboard']` 접두사로 전체 invalidate

### KPI 4 (인기 호출 앱) 행 클릭

- 표 행 클릭 → 같은 탭에서 `/app/{appId}/overview` 이동 (Dify 기본 모니터링 화면)
- 새 탭 / 모달 전환 없음. 브라우저 뒤로가기로 복귀 가능
- 키보드: 행 포커스 후 Enter (행에 `role="link"` + `tabIndex={0}`)

### "부서원" 메트릭 정의 (KPI 2 표 전용)

- 표의 "부서원" 컬럼 = `COUNT(DISTINCT actor_id)` (audit 마트). actor_type 무관(콘솔 사용자 + end_user 합산)
- 부서 단위 메트릭이므로 actor가 해당 부서 구성원인 경우만 카운트 (RBAC JOIN)
- KPI 3 (calls) 표는 부서원 컬럼 없음 — `호출`·`Top 호출 앱`으로 정보 보완. KPI 4 (apps) 표는 앱 그레인이라 `이용자` 컬럼(`COUNT(DISTINCT actor_id)` 동일 정의, 명칭만 다름)

### "부서원당 이용 앱 수" 메트릭 정의 (KPI 2 표 전용)

- 분자 = `COUNT(DISTINCT target_app_id)` WHERE actor_dept_id = 부서 (= 좌하 차트 막대값)
- 분모 = `COUNT(DISTINCT actor_id)` WHERE actor_dept_id = 부서 (= 같은 표 "부서원" 컬럼)
- 표시: 1자리 소수점 (예: `0.5`, `2.3`). 분모 0 → "-" (H-DASH-07 패턴 준용)
- 의미: 부서 내 1인당 평균 이용 앱 수. "이용의 폭이 부서 사이즈 정규화된 강도"

### "Top 이용 앱" 컬럼 정의 (KPI 2 표 전용)

- 부서 내 호출 수 1위 앱의 이름
- SQL: `SELECT DISTINCT ON (actor_dept_id) target_app_id, COUNT(*) FROM ... GROUP BY actor_dept_id, target_app_id ORDER BY actor_dept_id, COUNT(*) DESC`
- 앱명: `apps.name` JOIN (audit details `targetAppName` fallback)
- 동률 시: target_app_id 알파벳/UUID 사전순 첫 번째 (DISTINCT ON 자연 결과)

### "Top 호출 앱" 컬럼 정의 (KPI 3 표 전용)

- 부서 소유(owner) 앱 중 기간 내 호출 수 1위 앱의 이름
- 분자 기준 = `SUM(spx_mv_kpi_calls_daily.calls)` per (owner_dept_id, target_app_id)
- SQL: `ROW_NUMBER() OVER (PARTITION BY app_owner_dept_id ORDER BY SUM(calls) DESC)` = 1 행만 추출 (Layer 2 mart 활용)
- 앱명: `apps.name` JOIN. 앱 삭제 시 `"삭제된 앱"` fallback
- KPI 2 "Top 이용 앱"(actor_dept_id 기준 = 부서 구성원의 이용 1위 앱)과 의미 다름. KPI 3은 owner_dept_id 기준 = 소유 부서의 호출 1위 앱

### "RPS" 컬럼 정의 (5/20 신설 — KPI 3 표 전용)

- 기간 평균 RPS = `SUM(calls) / period_seconds` (분모 = (end - start) 초 단위, 최소 1)
- 표시: 소수점 2자리 (예: `0.13`, `2.45`)
- 분모 0 회피 = `max(period_seconds, 1)` 가드
- 좌하 차트(절대 호출 수)와의 정보 분리: 부서 사이즈/기간 정규화된 부하 강도

### "추세" 컬럼 정의 (KPI 2 · KPI 3 표 공통)

- KPI 2 = 이용 앱 수의 전기간 대비 증감 %
- KPI 3 = 호출 수의 전기간 대비 증감 %
- 분자 = 현 기간 지표 - 이전 기간 지표
- 분모 = 이전 기간 지표
- 표시: KPI 카드 증감 뱃지 패턴 준용 (▴ / ▾ / 0% / +신규 / - / N/A — `hdd/specs/requirements/kpi-cards.md § 증감 뱃지` 동일)
- 분모 0 → "+신규" / "-" 분기 (H-DASH-07)
- 이전 기간 자동 산출: 현 기간과 동일 길이 직전 (예: 4/1~4/30 선택 시 이전 = 3/2~3/31)

### "마지막 사용" 컬럼

- `MAX(occurred_at)` (audit 마트의 가장 최근 이벤트 시각). 표시 형식은 "N분 전 / N시간 전 / N일 전" 상대 시간
- KPI 4 (apps) 표에서만 사용. KPI 2 (users) 표·KPI 3 (calls) 표는 "추세"로 대체

### PM 확인 항목 (보류 결정)

- **dept-new-creations "신규" 정의**: ownership 등록 시점 기준 vs Dify 앱 생성 시점 기준. PM 확정 후 쿼리 작성
- **api_call (nginx) 부서별 분류 가능성**: 현재 audit가 모든 호출을 캡처하지 못할 가능성. nginx 로그 분류 필요성 PM 확인
- **dataset_operator 권한 노출 부재**: 설정 모달 등 UI에서 dataset_operator role 노출 안 됨. 실제 사용자 존재 여부 PM 확인 후 별도 정책 보강 검토
- 선택 후보 컬럼 (응답시간 / 만족도 / TPS / 앱 타입): 후보로만 표기, 본 단계 audit 검증 작업 없음

## Harness 적용 / 후보

### 적용 Harness

| ID | 적용 위치 |
|---|---|
| H-DASH-04 | 모든 부서별 차트·표의 미배정 fallback |
| H-DASH-07 | 증감률 분모 0 처리 (표의 추세 / 호출 변동) |
| H-DASH-18 | 에러 분류 ILIKE 6종 (audit `details->>'error'`) |
| H-DASH-19 | 모달 마운트 제약 폐기 — ESC / `<h1>` / URL state 자유 사용 |
| H-DASH-20 | 차트 드로어 보류 — 좌하·우하 차트 클릭 인터랙션 없음 |

### Harness 후보 (미검증)

| 후보 코드 | 가설 결함 | 검증 방법 | 출처 강도 |
|----|------|------|---|
| DRILL-LAYOUT-DRIFT | 메트릭 전환 시 차트 위치 / 영역 높이 / 색상 체계가 변경됨 (PDF 명시 제약 위반) | 메트릭 전환 후 컨테이너 그리드 / 색상 토큰 / 고정 높이 시각 1:1 대조 | 🟢 강함 — PDF v0.3 17p + 5/13 고정 높이 전역 정책 |
| H-CAND-audit-appmode-missing | audit `details`에 AppMode 미보존 시 토큰/호출 분류 불가 | collector 보강 후 `SELECT DISTINCT details->>'appMode' FROM audit_events` 결과 검증 | 🟡 중간 |
| H-CAND-audit-wf-debug-filter-missing | audit `details`에 `triggeredFrom` 미보존 시 워크플로 디버깅 필터 불가 | 동상 | 🟡 중간 |

> 구현/테스트 단계에서 실제 결함이 발생하거나 소스 분석으로 검증되면 `defect-catalog.md`에 정식 ID 부여.

## 기존 컴포넌트 영향

- `kpi-cards`: 카드에 `isActive`, `onActivate`, `metricKey` prop 추가
- `dept-objects`, `model-tokens`, `dept-activity`: 종합 뷰에서 현 동작 유지
- 활성 모드의 차트·표는 별도 컴포넌트로 분리 (`drill-charts/`, `drill-tables/`) — 메트릭별 데이터 모델 / 컬럼이 다르므로 단순 prop 분기보다 명확

## 관련 노트

- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|KPI 카드 requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md|Design]]
