---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: KPI 카드 4종
harness: [H-DASH-01, H-DASH-03, H-DASH-04, H-DASH-05, H-DASH-07]
harness_candidates: [H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
date: 2026-04-28
last_updated: 2026-05-26
---
# KPI 카드 4종 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md|Design]]
> 순서: 프론트엔드(목업) → 백엔드 → 연동 → 테스트

## 1단계: 프론트엔드 (목업 데이터)

### 1-A. 카드 컴포넌트 / 레이아웃

- [ ] `kpi-card.tsx` — 제목 + (?) 툴팁 + 기간 라벨 + 큰 숫자 + 증감 뱃지 + subtitle/부가정보 슬롯
- [ ] `kpi-section.tsx` — 4카드 그리드, **고정 높이**(`h-[140px]` 등), 데이터 양에 카드 크기 비변동 검증
- [ ] `diff-badge.tsx` — 6가지 분기(`+신규` / `-` / `N/A` / 0% / 정상 % 상승·하락) 구현 (H-DASH-07, H-DASH-05)
- [ ] `format-number.ts` — `formatCount` (`<10K` raw, `≥10K` K/M 압축) + `formatExact` (툴팁용)
- [ ] 큰 숫자 hover 툴팁 — `≥10K`일 때 정확 raw 값 표시
- [ ] (?) 아이콘 + 카드별 툴팁 문구 (requirements 표 참조)
- [ ] subtitle 영역 — KPI 4 앱명, `truncate` + `title=` hover 풀텍스트
- [ ] active 상태 시각 강조 (테두리 + 살짝 강조 배경) — drill-through 진입 표시

### 1-B. 4축 KPI 카드 세트로 갱신

- [ ] KPI 1 (총 오브젝트): mock에 `new_count`(메인) + `count`(보조) + App/지식/도구 분해 (분해는 dept-objects 범례로 이동 — KPI 카드 내부 표시 없음)
- [ ] KPI 2 (총 이용 앱 수): mock에 단일 count + 기간 내 호출 0 케이스(count=0, diff_label="N/A") 포함
- [ ] KPI 3 (총 앱 호출량): mock에 count + 증감 뱃지 (RPS 평균은 KPI 카드 내부 표시 없음 — drill-through 표로 이전)
- [ ] KPI 4 (인기 호출 앱): mock에 `count`(호출 수) + `app_name` subtitle(큰 자리에 truncate) + Top 앱 없음 케이스(`app_id=null`, `diff_label="N/A"`) 포함
- [ ] **이전 4번째 카드(에러율) 잔존 코드/타입/필드 제거** — `error_rate`, `valueSuffix`, `invertColor`, `diff_pp`, 24h 라벨 등

### 1-C. URL 상태 / 컨트롤 연동

- [ ] `dashboard-controls` → URL query string으로 `start`/`end`/`period` 갱신 → KPI 섹션은 query string에서 읽음
- [ ] TanStack Query `queryKey` 표준화 — `['dashboard', 'kpi', params]`
- [ ] 페이지 헤더 새로고침 버튼 클릭 시 `invalidateQueries({ queryKey: ['dashboard'] })` 동작 확인

## 2단계: 백엔드

- [ ] SQLAlchemy 모델 import 확인 (`apps`, `audit_events`, `resource_ownership`, `department_members`)
- [ ] `DashboardKpiService` 클래스 생성 (`api/services/admin/`)
  - [ ] `get_total_objects()` — `resource_ownership` COUNT + 미배정 fallback (H-DASH-04)
  - [ ] `get_dept_adopted_apps(current_user)` — `COUNT(DISTINCT details->>'targetAppId')` for dept member actors. 부서 미소속 → 0 + "N/A"
  - [ ] `get_api_calls()` — audit_events COUNT + AppMode 분기 (H-DASH-01) + 디버그 필터 (H-DASH-03), `rps_avg` 계산
  - [ ] `get_top_app_calls()` — GROUP BY targetAppId ORDER BY count DESC LIMIT 1, AppMode 분기, 디버그 필터
  - [ ] `calc_diff()` — 증감률 유틸 (H-DASH-07)
  - [ ] `calc_diff_with_history_check()` — 이전 기간 데이터 없음 → "N/A" (H-DASH-05)
- [ ] AppMode 분기 — `details->>'appMode'` 우선, 미수집 시 `apps.mode` JOIN fallback (H-CAND-audit-appmode-missing)
- [ ] workflow 계열 디버그 필터 — `details->>'triggeredFrom'` 미수집 가능성 표본 카운트 검증 (H-CAND-audit-wf-debug-filter-missing)
- [ ] API 엔드포인트 등록 — `GET /console/api/dashboard/kpi`
  - [ ] Blueprint: `api/controllers/console/dashboard/` 아래 등록 (admin prefix 폐기 반영)
  - [ ] 권한 데코레이터: 로그인만 (기존 admin 전용 데코레이터 제거)
- [ ] Pydantic 스키마 (`dashboard_kpi_schemas.py`) — `KpiResponse`/`TotalObjects`/`DeptAdoptedApps`/`ApiCalls`/`TopAppCalls`

## 3단계: 연동

- [ ] `use-dashboard-kpi.ts` 훅 — 목업 데이터를 실 API 호출로 교체, `staleTime: 5분`
- [ ] 실 데이터로 화면 검증
- [ ] 페이지 헤더 새로고침 버튼 일괄 invalidate 동작 확인 (`['dashboard']` prefix)
- [ ] KPI 4 카드 클릭 → drill-through 활성화 검증 (kpi-drill-through spec과 연계)
- [ ] KPI 4 drill-through 표 행 클릭 → `/app/{appId}/overview` 같은 탭 이동 검증
- [ ] KPI 1·2·3 표 행 클릭 → 동작 없음 확인 (회귀 방지)

## 4단계: 테스트

- [ ] H-DASH-01: ADVANCED_CHAT 앱의 호출이 KPI 3·4에서 1번만 카운트 (messages 계열만)
- [ ] H-DASH-03: 디버깅 실행(`invokeFrom='debugger'` / `triggeredFrom='debugging'`)이 모든 KPI에서 제외
- [ ] H-DASH-04: `resource_ownership`에 없는 앱이 KPI 1에 "미배정"으로 포함되고 KPI 4 표 6컬럼 "부서(소유)"에 "미배정" 표시
- [ ] H-DASH-05: 이전 기간 데이터 없을 때 모든 KPI 증감률 "N/A"
- [ ] H-DASH-07: 이전 = 0, 현재 > 0 → "+신규"; 둘 다 0 → "-"
- [ ] KPI 2: 부서 미소속 사용자 케이스 → count=0 + "N/A"
- [ ] KPI 2: 다중 부서 소속 사용자 합집합 계산 정확성
- [ ] KPI 4: 기간 내 호출 0건 → `app_id=null` + "N/A" 표기, 카드 클릭 시 drill-through 정상 비활성 또는 안내
- [ ] KPI 4 subtitle 앱명 truncate + hover 풀텍스트 정상 동작
- [ ] **고정 높이 회귀**: subtitle/부가정보 데이터 양이 늘어도 4 카드 동일 높이 유지
- [ ] H-CAND-audit-appmode-missing: collector 보강 전 fallback(apps JOIN) 경로 동작 검증
- [ ] H-CAND-audit-wf-debug-filter-missing: workflow 디버그 이벤트 표본 카운트로 필터 누락 여부 측정

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md|Design]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|kpi-drill-through Requirements]] (KPI 4 클릭 후 화면)
