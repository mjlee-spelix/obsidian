---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: 컨텍스트 바 (Phase 2 동적 인터랙션)
phase: 2
status: "🔒 보류 (2026-05-22)"
harness: [H-DASH-21]
harness_candidates: [CTXBAR-CHIP-DISMISS-LOST]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_pdf: "0. Inbox/화면설계_v0.3.pdf"
date: 2026-05-04
last_updated: 2026-05-26
---

> 🔒 **보류 (2026-05-22)** — chart-drawer 5/13 보류 동일 패턴
>
> `page.tsx`에서 `ContextBar` 렌더링만 비활성. 컴포넌트 구현체(`web/app/components/admin/context-bar/*`) 코드 보존. ESC 해제 동작은 현재 KPI 카드 토글에 통합 (`useKpiDrillThrough` 훅이 ESC 일괄 처리).
> 본 spec 본문은 design intent base 참고용으로 보존. 현재 구현 대상 아님.
> 참조: SESSION_HISTORY 2026-05-22 § 차트/표 UI 개선 / `chart-drawer` (5/13 보류) 동일 패턴. 재도입 가능성 열림 (PM 결정 동반 트리거). 결함 박제: H-DASH-21.

# 컨텍스트 바 — Requirements

> Phase 2 동적 인터랙션 #3 — 활성 필터 상태(drill-through 메트릭)를 칩으로 한 줄에 모아 표시. drill-through 활성 모드 시에만 노출.
>
> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/context-bar.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/context-bar.md|Tasks]] · [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|drill-through]]

## 범위 (Slim 컨텍스트 바)

본 spec은 **drill-through 활성 메트릭만 노출하는 slim 컨텍스트 바**.

| 항목 | 처리 | 사유 |
|---|---|---|
| 기간 칩 | ❌ 제외 | `DashboardControls`(페이지 헤더 슬롯)가 이미 기간 표시 + 변경 — 중복 회피 |
| URL 북마크 | ❌ 제외 | `/dashboard` 라우트는 query string으로 기간/메트릭 상태가 이미 deep link화 — 별도 버튼 불필요 |
| 새로고침 | ❌ 제외 | `DashboardControls`에 이미 새로고침 버튼 있음 — 중복 회피 |
| 5분 캐시 안내 | ⏳ 보류 | DashboardControls 호버 툴팁으로 흡수 검토 (별도 작업) |
| **활성 메트릭 칩** | ✅ 채택 | drill-through active 상태 표시 + 해제 경로 |
| 활성 부서 칩 | ❌ 제외 | drill-through 활성 시 메트릭 그레인(부서/앱)이 차트·표에 이미 명시 — 중복 회피 |

## 화면 요구사항

### 노출 조건

- `activeMetric === null` (종합 뷰): **컨텍스트 바 미렌더** (자리도 차지 안 함)
- `activeMetric !== null` (drill-through 활성): 컨텍스트 바 노출

### 위치

- KPI 섹션 바로 아래, 차트 영역 위
- 페이지 컨텍스트 영역 (좌측 정렬, 한 줄 높이)
- KPI 섹션·차트 영역과 같은 좌우 패딩 정렬

### 표시 내용

```
[메트릭: API 호출 ×]   ← drill-through 활성 메트릭 칩
```

- **칩 라벨**: `메트릭: {메트릭명}` (예: "메트릭: API 호출")
- **× 닫기 아이콘**: 클릭 시 drill-through 해제 (`toggle(activeMetric)` → null)
- 칩 hover: 살짝 강조, "클릭하여 종합 뷰로" 툴팁

### 메트릭 라벨 매핑

| `activeMetric` | 칩 라벨 | 그레인(컨텍스트 단위) |
|---|---|---|
| `objects` | "메트릭: 총 오브젝트" | 부서명 |
| `users` | "메트릭: 사용자 수" | 부서명 |
| `calls` | "메트릭: API 호출" | 부서명 |
| `apps` | "메트릭: 앱별 통계" | 앱명 |

- 4종 메트릭 = `users / calls / apps / objects`
- 칩 라벨은 [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|kpi-cards]] 카드 제목과 일치시킴
- **그레인 의미**: drill-through 활성 시 차트·표가 보여주는 행 단위(부서명 행 vs 앱명 행)와 일치. 본 칩은 그레인 정보 자체를 직접 노출하지는 않으나, 메트릭 라벨이 그 단위를 암시함.

### 칩 시각

- **컨테이너**: `inline-flex` 한 줄
- **칩 디자인**: 기존 칩 패턴 차용 (`web/app/components/base/badge` 또는 동급)
  - 기존 토큰/컴포넌트 우선 규칙 따름 — 새 칩 디자인 만들지 않음
- **× 아이콘**: `RiCloseLine` 또는 동급
- **enter/exit**: KPI 클릭 시 자연 등장 (drill-through state 변경 → 컨텍스트 바 마운트). 별도 애니메이션 불필요.

## 마운트 환경

- **페이지**: `/dashboard` 라우트 (`web/app/(commonLayout)/dashboard/page.tsx`)
- **상태**: `useKpiDrillThrough` 훅의 `activeMetric`에 100% 의존 (자체 state 없음)
- **소유자**: `DashboardPage` (drill-through 훅 결과를 prop으로 받음)
- **데이터 소스**: 별도 fetch 없음 — drill-through 메트릭 상태의 단방향 뷰
- **ESC 키**: 페이지 라우트 환경이라 ESC 자유롭게 활용 가능. 본 컴포넌트는 ESC를 가로채지 않고 drill-through 훅이 일괄 처리 (kpi-drill-through 참조)

## 키보드 접근성

| 키 | 동작 |
|---|---|
| `Tab` | 컨텍스트 바 칩 → 차트 영역 순회 (KPI 카드와 차트 사이 자연스러운 위치) |
| `Enter` / `Space` | 포커스된 칩의 × 활성화 (drill-through 해제) |

- 칩은 `role="button"` + `tabIndex={0}` + `aria-label="활성 메트릭 해제: {메트릭명}"`

## 비즈니스 규칙

### 칩 = drill-through state의 단방향 뷰

- 컨텍스트 바는 **표시 + 해제**만 담당, 메트릭 변경/추가 기능 없음
- 메트릭 변경은 KPI 카드 클릭으로 (drill-through 표준 경로)
- → 단순성 확보, KPI 카드와 책임 중복 회피

### 본 spec 범위 외 (향후 확장 후보)

- 추가 필터 도입 시 (예: 부서 필터, 모델 필터): 칩 추가
- 단, 본 Phase 2에서는 **활성 메트릭 칩 1종만**

## Harness 후보 (미검증)

| 후보 코드 | 가설 결함 | 검증 방법 | 출처 강도 |
|---|---|---|---|
| CTXBAR-CHIP-DISMISS-LOST | 칩 × 클릭이 drill-through state와 동기화 실패 (예: 칩에서만 사라지고 KPI 카드는 계속 active) | drill-through state를 단일 진실로 두고 컨텍스트 바는 파생 뷰만, 자체 state 금지 검증 | 🟢 강함 — 단방향 뷰 원칙 위반 패턴 |

## 기존 컴포넌트 영향

- **`kpi-drill-through` (Phase 2)**: `useKpiDrillThrough` 훅이 반환하는 `activeMetric`을 컨텍스트 바에 prop 전달 — 훅 변경 없음
- **`DashboardPage`**: 컨텍스트 바를 KPI 섹션과 차트 영역 사이에 렌더 (조건부)
- **`queryKey`**: 본 컴포넌트는 백엔드 호출 없음 — queryKey 영향 없음
- 기타 컴포넌트 영향 없음

## 관련 노트

- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|drill-through requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/dashboard-controls.md|dashboard-controls requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|kpi-cards requirements]]
