---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: 부서별 오브젝트 분포
harness: [H-DASH-04, H-DASH-13]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_image: "images/Dify/(Dify) 모니터링 - 챗봇.png"
current_implementation_image: "images/구현/(화면 구현) 부서별 차트 1차 0430.png"
date: 2026-04-29
last_updated: 2026-05-22
---
# 부서별 오브젝트 분포 — Requirements

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/dept-objects.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dept-objects.md|Tasks]]
> 데이터 소스 검증: [[3. 프로젝트/spx-agent/references/objects-charts-feasibility.md|objects-charts-feasibility.md]]

## 화면 요구사항

**수평 스택 막대 차트** (Y축 = 부서, X축 = 오브젝트 수). 색상 3개로 앱 / 지식 / 도구 구분.

```
┌─ 부서별 오브젝트 현황 ─── ● 앱 N  ● 지식 N  ● 도구 N ┐  ← 카드 헤더 React 범례 (클릭 시 필터)
│ IT 본부  ████████████████████  135                      │
│ 마케팅   █████████████          80                      │
│ 재무팀    ████████              55                      │
│ 인사팀    ██████                38                      │
│ 미배정    ████                   22                      │
└──────────────────────────────────────────────────────────┘
```

| 구성요소 | 설명 |
|---------|------|
| 수평 스택 막대 | 부서별 오브젝트 수 (resource_type별 3색, 좌→우 누적) |
| 부서명 | 막대 좌측 라벨 |
| 합계 숫자 | 막대 우측에 부서별 총 오브젝트 수 표시 |
| 카드 헤더 React 범례 | **카드 헤더 우측에 React로 직접 렌더** (ECharts 내장 범례 사용 안 함). 색상 dot + i18n 라벨 + 동적 개수. 클릭 시 해당 타입만 필터링(같은 항목 재클릭 → 전체 복귀) |
| "미배정" 막대 | `spx_resource_ownership`에 없거나 `owner_department_id IS NULL`인 레거시 오브젝트 (H-DASH-04). 화면 설계의 "외부" 라벨은 본 차트에서 "미배정"으로 통합 (별도 외부/end_user 분리 X) |
| 색상 (전역) | 앱=`util-colors-blue-blue-500`, 지식=`util-colors-teal-teal-500`, 도구=`util-colors-orange-orange-500` (KPI 카드 부가 정보와 통일) |

> 화면 방향: Dify 모니터링 차트 패턴이 모두 수평이고, 부서명이 길어질 수 있어 수평 막대가 자연스러움.

## 카드 헤더 React 범례 (5/22 신설)

- **위치**: 카드 헤더 우측 (`title` 옆) — ECharts 차트 본문 외부
- **렌더 형식**: 타입별로 `<button>` 1개씩 (총 3개) — 클릭 가능
  - 색상 dot (`h-2 w-2 rounded-full` + `style.backgroundColor`)
  - i18n 라벨 (`common.objectType.app` / `.kb` / `.tool` → 앱/지식/도구)
  - 동적 개수 (해당 타입 누적 합계, `formatCount` 적용)
- **클릭 동작**:
  - 클릭한 타입만 차트에 노출 (다른 타입 series는 zero-out)
  - 같은 타입 재클릭 → 필터 해제, 전체 복귀
  - 비활성 범례 항목은 `opacity-40`으로 시각 dim
- **상태**: 카드 컴포넌트 내부 `useState<filter>` (URL state 미사용 — 차트 단독 시각 필터)
- ECharts 내장 `legend` 옵션은 사용하지 않음(범례 자체 제거). 색상 dot/라벨/개수는 한국어 라벨 컨트롤이 필요하고 클릭 필터 정합도 React 쪽이 자연스러움

## 레이아웃

- **고정 높이 + 내부 스크롤** (전역 정책 — 부서 수가 늘어도 영역 크기 불변)
- 부서 수 ≥ 임계치(예: 8개)이면 차트 영역 내부에서 세로 스크롤
- 데이터 양에 따라 차트 카드 자체 크기가 변동되지 않음

## 기간/필터

- **기간 무관** — 오브젝트는 누적(state)이므로 현재 시점 기준
- `DashboardControls`의 기간 선택과 독립적으로 동작 (단, `dept-new-creations`(아래) 보조 뷰는 기간 필터 적용)

## 인터랙션

- **막대 클릭 = 동작 없음** — 차트 드로어는 보류 (H-DASH-20). 차트만 표시.
- **표 행 클릭 없음** — 본 컴포넌트는 차트 단독. 표 행 클릭 인터랙션은 KPI 4(인기 호출 앱)만 적용되며 본 차트와 무관.
- **카드 헤더 React 범례 클릭** = 해당 타입만 차트에 노출 (필터링) / 재클릭 시 해제. 동작 상세는 § 카드 헤더 React 범례 참조.

## 비즈니스 규칙

### 본 차트: 부서별 누적 오브젝트 (dept-cumulative)

- 입력 테이블: `spx_resource_ownership` + `spx_departments` 직접 조회
  - **audit 사용 안 함** — 오브젝트 보유는 "현재 상태(state)"이지 이벤트가 아님 (마트 입력 규칙의 예외, `.claude/CLAUDE.md` 아키텍처 불변식 #6)
- `owner_department_id`별, `resource_type`별 COUNT
- 부서는 **플랫 구조** (parent_id 트리 무시, 각 부서 독립 집계)
- 부서 기준 = **오브젝트 소유 부서 (owner_department_id)** — 본 차트는 오브젝트 자체가 행이므로 "누가 호출했나"(actor)는 무관

### 미배정 처리 (H-DASH-04 — 강화)

- `spx_resource_ownership.owner_department_id IS NULL` → COALESCE로 "미배정" 라벨
- 그러나 **레거시 앱이 `spx_resource_ownership`에 행 자체가 없는 경우 차트에서 완전 누락됨**
- → 운영 쿼리는 **`apps LEFT JOIN spx_resource_ownership`** 패턴 사용 권장 (누락 0건 보장, design.md 참조)
- 차트의 모든 막대 합계 = KPI 카드 "총 오브젝트" 수치와 일치해야 함

### 보조 뷰 (선택 — 같은 데이터 소스)

같은 `spx_resource_ownership`을 기반으로 두 가지 보조 집계가 가능. 본 컴포넌트에서 함께 구현할지 별도 위젯으로 분리할지는 차후 결정(현재는 디자인에 SQL 윤곽만 보존).

| 뷰 | 키 컬럼 | 비고 |
|---|---|---|
| top-owners (Top N 소유자) | `owner_account_id` | `created_by` 사용 금지 — 매핑률 29% vs `owner_account_id` 94.7%. 의미상으로도 "소유자"가 맞음 |
| dept-new-creations (부서별 신규 생성) | `created_at` | **PM 확인 후보**: `created_at`의 의미가 "소유권 등록 시점"(현재 구현 가능 범위) vs "앱 생성 시점"(apps JOIN 필요). 기본은 소유권 등록 시점 |

## 방어할 Harness

| ID | 결함 | 이 화면에서의 방어 |
|----|------|---------------|
| H-DASH-04 | 레거시 앱 미배정 누락 | `apps LEFT JOIN spx_resource_ownership` 패턴 + COALESCE 미배정 라벨 (단순 `spx_resource_ownership` 단독 GROUP BY는 운영에서 누락 발생) |
| H-DASH-13 | RBAC 스키마 변경 영향 | SQLAlchemy 모델 클래스로 접근, 컬럼명 직접 사용 금지 |
| H-DASH-20 | 차트 드로어 보류 | 막대 클릭 핸들러를 추가하지 않음 (Phase 2 인터랙션 미진입) |

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]] — H-DASH-04 / H-DASH-20
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|RBAC 스키마]] (회사 표준 `spx_` 접두사)
- [[3. 프로젝트/spx-agent/references/objects-charts-feasibility.md|오브젝트 차트 RBAC 단독 구현 가능성]]
