---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: KPI 카드 4종
harness: [H-DASH-01, H-DASH-03, H-DASH-04, H-DASH-05, H-DASH-07]
harness_candidates: [H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_image: "images/Dify/(Dify) 모니터링 - 챗봇.png"
current_implementation_image: "images/구현/(화면 구현) KPI 지표 2차 0430.png"
date: 2026-04-28
last_updated: 2026-05-22
---
# KPI 카드 4종 — Requirements

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/kpi-cards.md|Tasks]]
> 디자인 참고: Dify 모니터링 화면 (`images/Dify/(Dify) 모니터링 - 챗봇.png`)

## 화면 요구사항

페이지 상단에 4개 카드를 한 줄로 배치 (`/dashboard` 톱레벨 라우트 페이지 헤더 바로 아래). 각 카드는:
- **주 수치** (큰 숫자)
- **증감률** (이전 기간 대비 %, 상승 초록 / 하락 빨강 / 변동 없음 회색)
- **부가 정보** (카드별 상이)

### 4-Axis (네 가지 관찰 축)

| # | 축 | 카드 명 | 의미 |
|---|----|--------|------|
| 1 | Volume | 총 오브젝트 | 자산 누적량 — 기간 내 신규 등록 수 |
| 2 | Breadth | 총 이용 앱 수 | 채택 폭 — 기간 내 실제로 호출된 고유 앱 수 |
| 3 | Depth | 총 앱 호출량 | 사용 강도 — 기간 내 전체 앱 호출 합계 |
| 4 | Application Analysis | 인기 호출 앱 | 가장 많이 호출된 앱 1종 — 앱명 + 호출 수 |

| 카드 | 주 수치 | 우측 보조 표시 |
|------|--------|----------|
| 총 오브젝트 | 기간 내 신규 등록 수 (`new_count`) | 증감 뱃지 (전기간 대비 `diff_percent`) |
| 총 이용 앱 수 | 부서/전체 기준 기간 내 호출된 고유 앱 수 | 증감 뱃지 |
| 총 앱 호출량 | 기간 내 전체 앱 호출 수 | 증감 뱃지 |
| 인기 호출 앱 | 앱명 (truncate, hover 시 풀텍스트) | 호출 수 (뱃지 자리에 회색 배지로 표시) |

> "이용" = 앱 호출/소비 (소비자 관점) / "사용" = 플랫폼 전반 활동(생성 + 이용 포함). KPI 2·3·4는 모두 "이용" 차원, KPI 1은 "생성" 차원.
>
> KPI 4는 카드 head 큰 자리에 **앱명(subtitle)**을 표시하고 우측 뱃지 슬롯에 **호출 수**를 회색 배지로 표시. 클릭 시 KPI 4 drill-through(아래 § KPI 4 drill-through 레이아웃)로 전환.

## 마운트 환경 / 접근 권한

- 마운트: `/dashboard` 톱레벨 라우트 (`web/app/(commonLayout)/dashboard/page.tsx`) — 페이지 헤더 슬롯 아래 첫 영역
- 접근 권한: **로그인 사용자 전체** (`AppInitializer` 가드만 적용, 별도 RoleRouteGuard 없음)
  - dataset_operator 권한도 접근 가능 — UI에 dataset_operator 노출 부재(설정 모달 등)로 실제 사용 빈도는 PM 확인 후 정책 보강 검토 (PM 확정 필요)
- 자체 `<h1>` 사용 가능 — 모달 제약 없음
- 상태는 URL query string으로 — Phase 2 drill-through deep link 자연스러움

## 기간 선택

- `dashboard-controls`(페이지 헤더 슬롯) 기간 선택기에서 받은 `start`, `end`, `label` 파라미터 사용 (컨트롤 spec: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dashboard-controls.md|dashboard-controls requirements]])
- **상태 관리**: URL query string 단일 진실 (`?period=last7d` 또는 `?start=...&end=...`) — KPI 섹션은 query string에서 직접 읽거나 page-level 훅이 파싱한 값을 prop으로 받음
- 증감률 계산을 위해 **이전 기간**을 자동 산출
  - 예: 선택 기간이 4/1~4/30이면 이전 기간은 3/2~3/31 (동일 길이)

## 표시 규칙 (UI)

> Dify 모니터링 화면 패턴(`images/Dify/(Dify) 모니터링 - 챗봇.png`) 채택.

### 카드 레이아웃 (좌우 분리)

카드 내부는 가로 2열 레이아웃 (`flex items-start justify-between`):

```
┌──────────────────────────────────────────────────────┐
│ 총 오브젝트 (?)                              47     │
│ 지난 7 일                              ▴ +12.5%     │
└──────────────────────────────────────────────────────┘
  └─ 좌 컬럼 (제목+툴팁 / 기간 라벨)       └─ 우 컬럼 (큰 숫자 / 증감 뱃지)
```

- **좌 컬럼**: 위쪽에 제목(`system-sm-semibold text-text-secondary`) + (?) 툴팁 아이콘, 아래쪽에 기간 라벨(`system-2xs-regular text-text-tertiary`)
- **우 컬럼** (`flex flex-col items-end max-w-[55%]`): 위쪽에 큰 숫자(`text-2xl font-semibold text-text-primary`) 또는 KPI 4의 앱명(truncate + hover 툴팁), 아래쪽에 증감 뱃지/호출 수 배지
- 우 컬럼은 `max-w-[55%]` 제약으로 KPI 4 앱명이 좌 컬럼을 침범하지 않음

### 고정 높이 정책

- 4개 카드 + 하단 영역 모두 **고정 높이**. 데이터 양(부가 정보 줄 수, subtitle 길이)에 따라 카드 크기 변동 금지.
- subtitle/부가 정보가 컨테이너 폭을 넘으면 ellipsis(`truncate`) + hover 시 툴팁으로 풀 텍스트.
- 카드 내부 콘텐츠가 넘치면 카드를 키우지 않고 **영역 내부 스크롤** (단 카드 영역에 스크롤 발생 시 디자인 재검토).

### 폰트 스타일 토큰 표준

| 용도 | 토큰 |
|------|------|
| 카드 제목 | `system-sm-semibold text-text-secondary` |
| 기간 라벨 | `system-2xs-regular text-text-tertiary` |
| 주 수치 | `text-2xl font-semibold text-text-primary` |
| 증감 뱃지 | `system-xs-medium` + 색상 (success/destructive/tertiary) |
| KPI 4 앱명 (큰 자리) | `text-2xl font-semibold text-text-primary block max-w-full truncate` + hover 풀텍스트 툴팁 |
| KPI 4 호출 수 (뱃지 자리) | `system-xs-medium text-text-tertiary` (회색 배지) |

### 카드 컨테이너 스타일

- **모든 카드 동일한 회색 배경**, 카드 자체에 색상 구분 없음
- 기본 컨테이너: `rounded-xl border border-solid border-components-card-border bg-components-card-bg px-6 py-4 shadow-sm transition-all duration-200 ease-in-out`
- 다크/라이트 모드 자동 대응 (디자인 토큰 사용)

### 활성(active) 카드 시각 강조

drill-through 활성 카드는 다음 조합만 적용 — 배경 색상 변경 없음:

- `border-transparent ring-inset ring-2 ring-components-button-primary-bg-hover shadow-lg`
- `ring-inset`으로 카드 안쪽 ring → 부모 패딩과 무관 + 레이아웃 변동 0
- 기본 배경(`bg-components-card-bg`) 유지 — active용 별도 배경 토큰 미사용

### 제목 + 정보 아이콘 (?)

- 제목 우측에 작은 회색 (?) 아이콘 (`RiQuestionLine` 등)
- hover 시 툴팁으로 카드 의미 설명
- 툴팁 문구 (코드 기준 — 사용자 관점):
  | 카드 | 툴팁 |
  |------|------|
  | 총 오브젝트 | 선택 기간 동안 새로 만들어진 앱·지식·도구 수 |
  | 총 이용 앱 수 | 기간 내 1회 이상 사용된 앱의 수 |
  | 총 앱 호출량 | 전체 앱의 호출 횟수 합계 (디버그 제외) |
  | 인기 호출 앱 | 기간 내 가장 많이 호출된 앱 |

### 기간 라벨

- 제목 바로 아래 작은 회색 텍스트로 현재 선택 기간 표시
- 형식: `dashboard-controls` 기간 선택기의 라벨과 동일 (예: "지난 7 일", "지난 30 일")
- Dify와 동일하게 띄어쓰기 (`지난 N 일`)
- 모든 카드가 동일한 사용자 `period` 라벨을 사용 (카드별 별도 기간 고정 없음)

### 숫자 포맷

- `< 10,000` → raw + 천단위 콤마 (예: `47`, `128`, `9,820`)
- `≥ 10,000` → K/M 압축 (예: `15.4K`, `2.8M`)
  - 1자리 소수점 (예: `15.4K`, 단 `15.0K` 같은 경우 정수로 `15K`)
- **hover 시 정확한 raw 값 툴팁** (예: `15.4K` hover → `15,420`)
- 유틸: `@/utils/format`의 `formatNumber` 재사용 또는 확장

### 증감 뱃지

- 큰 숫자 아래(우 컬럼 두 번째 줄)에 표시
- 분기:
  | 조건 | 표시 | 색상 토큰 |
  |------|------|---------|
  | `diff_percent > 0` | ▴ +12.5% | `text-text-success` |
  | `diff_percent < 0` | ▾ -8.3% | `text-text-destructive` |
  | `diff_percent === 0` | - | `text-text-tertiary` |
  | `diff_label === "+신규"` (총 오브젝트, `newCount` 동반) | +신규 N (N=`new_count`) | `text-text-success` |
  | `diff_label === "+신규"` (그 외) | +신규 | `text-text-success` |
  | `diff_label === "-"` | - | `text-text-tertiary` |
  | `diff_label === "N/A"` 또는 `diff_percent === null && !diff_label` | N/A | `text-text-tertiary` |
  | 기타 임의 `diff_label` 문자열 (KPI 4 호출 수) | `diff_label` 그대로 (회색 배지) | `text-text-tertiary` |
- 화살표 아이콘: `▴` / `▾` 유니코드 문자 그대로 렌더 (Remix Icon 미사용)

### App / 지식 / 도구 분해 표시

- KPI 1 카드 내부에는 분해 정보 표시 **없음** — 좌우 분리 레이아웃 + 단일 주 수치(`new_count`) + 증감 뱃지만 노출
- App / 지식 / 도구 3종 카운트는 **부서별 오브젝트 차트의 카드 헤더 React 범례**로 이동 (색상 dot + i18n 라벨 + 동적 개수, 클릭 시 해당 타입 필터링)
  - 상세: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md|dept-objects requirements § 범례]]
- 색상 토큰은 차트와 통일 — App=`util-colors-blue-blue-500`, 지식(KB)=`util-colors-teal-teal-500`, 도구(Tool)=`util-colors-orange-orange-500`
- i18n key: `common.objectType.app` / `common.objectType.kb` / `common.objectType.tool` (한국어 = 앱/지식/도구, 영어 = App/KB/Tool)

## 클릭 인터랙션

| 영역 | 동작 |
|------|------|
| KPI 카드 1·2·3 클릭 | Phase 2 drill-through 활성화 (해당 KPI의 모드로 차트·표 메트릭 전환) |
| KPI 카드 4 클릭 | Phase 2 drill-through 활성화 — § KPI 4 drill-through 레이아웃 적용 |
| 좌하 차트 클릭 | **없음** (5/13 차트 드로어 보류 결정 — H-DASH-20) |
| 우하 차트 클릭 | **없음** (차트만 표시) |
| 하단 표 행 클릭 (KPI 1·2·3 모드) | **없음** |
| 하단 표 행 클릭 (KPI 4 모드) | 같은 탭으로 Dify 모니터링 페이지 이동 — `/app/{appId}/overview` |

> drill-through 진입/해제 메커니즘과 종합 뷰 복귀는 [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|kpi-drill-through requirements]] 단일 소스.

## KPI 4 drill-through 레이아웃 (인기 호출 앱)

KPI 4 카드 클릭 시 화면 하단 영역이 다음 4구역으로 전환:

```
┌──────────────────────┬──────────────────────┐
│ 좌상 카드             │  (KPI 1·2·3 영역은   │
│ 인기 호출 앱          │   비활성 / 흐림 처리) │
│ ─ 앱명 (큰 자리, trunc)│                      │
│ ─ 호출 수 (회색 배지)  │                      │
├──────────────────────┼──────────────────────┤
│ 좌하 차트             │ 우하 차트             │
│ 호출 수 Top10         │ 에러 발생 Top10       │
│ 가로 막대 (파란 그라)│ 가로 막대 (빨강 계열) │
├──────────────────────┴──────────────────────┤
│ 하단 표 (6컬럼) — 앱별 이용 현황              │
│ 앱 / 부서 / 호출 / 이용자 / 에러 / 마지막 사용 │
└──────────────────────────────────────────────┘
```

- **좌상 카드**: KPI 4 카드 자체가 active 상태로 강조
  - 타이틀 = `인기 호출 앱` 고정 (i18n: `common.kpi.topAppCalls`)
  - 큰 자리 = 앱명(subtitle), `max-w-[55%]` 안에서 `truncate` + hover 시 풀텍스트 툴팁
  - 우측 뱃지 슬롯 = 호출 수 (`formatCount(topApp.calls)`), 회색 배지
  - 좌측에는 카드 제목 + 기간 라벨만 표시 (큰 숫자 자리 자체가 앱명으로 대체됨)
- **좌하 차트**: `호출 수 Top10` (가로 막대). 색상은 파란색 그라데이션 (`util-colors-blue-blue-*` 단계). 클릭 동작 없음 — 차트만 표시
- **우하 차트**: `에러 발생 Top10` (가로 막대). 색상은 빨강 계열. 클릭 동작 없음
- **하단 표 6컬럼** (`앱별 이용 현황`):
  | 컬럼 | 데이터 |
  |------|--------|
  | 앱 | apps.name (audit details `targetAppName` fallback). 우상단에 모드 뱃지(Chat / Workflow / Completion) |
  | 부서 | resource_ownership → departments.name; 없으면 "미배정" (H-DASH-04) |
  | 호출 | 기간 내 호출 수 (audit_events 기반, AppMode 분기 적용) |
  | 이용자 | 기간 내 고유 사용자 수 (`COUNT(DISTINCT actor_id)`) |
  | 에러 | (실패 호출 / 전체 호출) × 100, 소수점 1자리. total=0 → "N/A" |
  | 마지막 사용 | 기간 내 최근 호출 시각 (`max(occurred_at)`) — 상대 시간 표시 |
- 표 행 클릭: 같은 탭으로 `/app/{appId}/overview` 이동 (Dify 기본 모니터링 페이지)
- 표 기본 정렬: 호출 내림차순 (서버 응답 순서 그대로). 부서명 가나다순 정렬 미적용(부서 그레인이 아니라 앱 그레인이라 컨벤션과 별도)

> 좌상 카드 외 KPI 1·2·3 카드는 비활성(흐림) 처리되며 클릭 시 해당 KPI 모드로 전환 가능.

## 데이터 소스 / 비즈니스 규칙

> **마트 입력 원칙** (아키텍처 불변식 #6): KPI 마트의 입력은 `audit_events` 단일 SoT + RBAC JOIN. 오브젝트 차트만 `resource_ownership` 직접 조회 (state라 audit 본질 불가).

**총 오브젝트 (Volume) — KPI 1:**
- `resource_ownership` 테이블 기준 COUNT, `object_type`별 분리
- **`new_count` = 기간 내 신규 등록 수 (메인 값)**:
  - `WHERE created_at BETWEEN start AND end`
  - "신규"의 정의: ownership 레코드의 `created_at` 기반. 앱 자체 생성 시각(`apps.created_at`)과 다를 수 있음 — PM 명시 컨펌 대기 (ownership reg vs app creation)
- **`count` = 기간 종료 시점에 살아있는 오브젝트 수** (응답 보조 필드, UI 카드에는 미표시):
  - `WHERE created_at <= end` 필터 — 기간 종료까지 등록 완료된 오브젝트
  - `app_count` / `kb_count` / `tool_count`도 동일 필터 적용 (응답 보조)
  - (soft delete 컬럼 추가 시 `AND (deleted_at IS NULL OR deleted_at >= start)` 추가 가능)
- 증감률: `curr_new` vs `prev_new` 비교 (이전 동일 길이 기간 대비 신규 등록 증감, `diff_percent`)
- `resource_ownership`에 없는 레거시 앱 → "미배정"으로 포함 (H-DASH-04)
- UI 표시:
  - 큰 숫자 = `new_count` (기간 내 신규 등록 수)
  - 증감 뱃지 = `▴/▾ N%` (`diff_percent`) / `+신규 N` (N=`new_count`, 이전=0 + 신규 발생) / `-` (이전=0 + 현재=0)
  - App / 지식(KB) / 도구(Tool) 분해는 KPI 카드 내부에 표시하지 않음 — 부서별 오브젝트 차트의 카드 헤더 React 범례에서 노출

**총 이용 앱 수 (Breadth) — KPI 2:**
- 정의: 기간 내 **실제로 호출된 고유 앱의 수**
- 메트릭: `COUNT(DISTINCT target_app_id)` from audit 마트 (디버그 제외)
- 데이터 소스: `spx_mv_audit_enriched` (action ∈ {message_send, workflow_execute, api_call …})
- 디버깅 실행 제외: `NOT is_debug` (audit 마트 derived 컬럼) + `is_canonical_call` (정규 호출만)
- UI 큰 숫자 = 기간 내 고유 앱 수, 증감 뱃지 = 전기간 대비 `diff_percent`

**총 앱 호출량 (Depth) — KPI 3:**
- 기간 내 전체 앱 호출 수 (audit 마트 기반)
- ADVANCED_CHAT은 messages 계열 이벤트만 카운트 — `workflow_runs` 계열 이벤트 제외 (H-DASH-01)
  - audit 마트의 `is_canonical_call` 컬럼이 AppMode 분기와 디버그 제외를 모두 합성 — `appMode` 직접 분기 미사용 가능. 단 collector `details->>'appMode'` 보강은 별도 유효성 확보 트랙 (H-CAND-audit-appmode-missing)
- 디버깅 실행 제외: `NOT is_debug` (H-DASH-03)
- nginx access log 기반 API 호출 부서별 분류 가능성은 **PM 명시 컨펌 대기** — 현 audit_events는 콘솔 actor가 명시되지만 외부 API key 호출은 `account_id` 매핑 부재 가능성

**인기 호출 앱 (Application Analysis) — KPI 4:**
- KPI 4 카드 자체는 **앱명(큰 자리) + 호출 수(뱃지 자리)** 만 표시. 카드 제목은 `인기 호출 앱`으로 고정
- Top 앱 선정: 기간 내 호출 수 최대 앱 — audit 마트 GROUP BY `target_app_id` ORDER BY count DESC LIMIT 1
- AppMode 분기 / 디버그 제외는 KPI 3과 동일 (`is_canonical_call`, `NOT is_debug`)
- 활성 모드(drill-through) 시 좌하·우하 차트·표는 동일 기준에서 LIMIT 확장 (호출 수 Top10 / 에러 발생 Top10 / 앱별 이용 현황 6컬럼)
- `count = 0` 이면 `app_id = null, app_name = null, diff_label = "N/A"`로 응답 → 카드는 앱명 자리에 placeholder, 뱃지 자리에 `N/A`

**증감률 공통:**
- 공식: `(현재 - 이전) / 이전 × 100`
- 이전 = 0, 현재 > 0 → "+신규" 표시 (H-DASH-07)
- 이전 = 0, 현재 = 0 → "-" 표시 (H-DASH-07)
- 이전 기간 데이터가 Celery Beat에 의해 삭제된 경우 → "N/A" 표시 (H-DASH-05)

## 방어할 Harness

| ID | 결함 | 이 화면에서의 방어 |
|----|------|---------------|
| H-DASH-01 | ADVANCED_CHAT 이중카운트 | 총 앱 호출량·인기 호출 앱: audit 마트의 `is_canonical_call`로 정규 호출만 카운트 |
| H-DASH-03 | 디버깅 혼입 | audit 마트의 `NOT is_debug` 필터 (derived 컬럼) |
| H-DASH-04 | 레거시 앱 미배정 | 총 오브젝트·KPI 4 표: LEFT JOIN + "미배정" fallback |
| H-DASH-05 | Celery Beat 삭제 | 이전 기간 데이터 없으면 증감률 "N/A" |
| H-DASH-07 | 증감률 NaN | 분모 0 → "+신규" / "-" 분기, KPI 4 에러율 분모 0 → "N/A" |

### Harness 후보 (미검증)

| ID | 결함 후보 | 영향 카드 |
|----|----------|---------|
| H-CAND-audit-appmode-missing | audit_events.details에 `appMode` 미수집 → 마트 `is_canonical_call` 산출 시 fallback 정확도 검증 필요 | 총 앱 호출량, 인기 호출 앱 |
| H-CAND-audit-wf-debug-filter-missing | workflow_runs 계열 audit 이벤트에 `triggeredFrom` 미수집 → 마트 `is_debug` 누락 가능성 | 총 앱 호출량, 인기 호출 앱 |

> 두 후보 모두 collector 보강(P0) 후 실데이터로 검증 → 패턴 등록 또는 회피로 결론.

## PM 명시 컨펌 대기 항목

- KPI 1 "신규" 정의: `resource_ownership.created_at`(소유권 등록 시점) vs `apps.created_at`(앱 생성 시점) — 어느 쪽이 사용자가 기대하는 "신규"인지
- KPI 2 "이용" 정의: 1회 이상 호출 (현 구현) vs DAU 등 더 엄격한 정의
- KPI 3 총 앱 호출량의 부서별 분류 가능성: nginx access log 기반 외부 API key 호출이 부서로 귀속 가능한지 (현 audit_events는 콘솔 actor 위주)
- 표 헤더 정렬/필터 정책: 전역 적용 여부 (KPI 4 표 6컬럼에 우선 적용 가능)
- dataset_operator 권한 사용자의 실제 사용 빈도 — 미사용이면 별도 가드 불필요, 사용자 있으면 KPI 가시성 정책 보강

## 관련 노트

- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit_events details 매트릭스]]
- [[3. 프로젝트/spx-agent/references/dify-app-modes.md|AppMode 규칙]]
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|RBAC 스키마]]
