---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: 차트 드로어 (Phase 2 동적 인터랙션)
phase: 2
status: "🔒 보류 (2026-05-13 이사님)"
harness: []  # 검증 후 등록 (5단계 테스트에서 선별)
harness_candidates: [DRAWER-CONTEXT-MISMATCH, DRAWER-CSV-EXPECTATION, DRAWER-FOCUS-TRAP]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_pdf: "0. Inbox/화면설계_v0.3.pdf"
reference_pdf_pages: "16"
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

# 차트 드로어 — Requirements

> Phase 2 동적 인터랙션 #2 — 부서별 차트 막대 클릭 시 우측 슬라이드 인 드로어가 열리며, 해당 부서의 메트릭 상세를 표시.
>
> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/chart-drawer.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/chart-drawer.md|Tasks]] · [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|drill-through]]
> 참조: PDF v0.3 16p

## 화면 요구사항

### 트리거 차트 (드로어 진입점)

다음 두 종류의 부서별 차트에서 막대 클릭 시 드로어 열림:

1. **종합 뷰의 부서별 오브젝트 차트** ([[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md|dept-objects]])
   - 컨텍스트: `metric: 'objects-overview'`, `department_id`, `department_name`
2. **drill-through 활성 모드의 부서별 차트들** ([[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|kpi-drill-through]])
   - 컨텍스트: `metric: 'calls' | 'users' | 'objects' | 'errors'`, `department_id`, `department_name`
   - 좌 차트(부서별 ___)에서만 발동, 우/하단 차트(모델별/Top)는 클릭 비활성

> ※ 모델별 차트(`model-tokens`)나 부서별 활동 표(`dept-activity`)에서의 진입은 본 spec 범위 외. 추후 확장 시 별도 결정.

### 드로어 레이아웃

```
┌──────────────────────────────────────────┐
│ [부서명+팀]·[메트릭]      [CSV] [×]      │  ← 헤더: 부서명(+팀) + 메트릭 칩 + CSV + 닫기
├──────────────────────────────────────────┤
│ [부서] [지표] [기간] [에러]              │  ← 컨텍스트 칩 4종 (표시 전용)
├──────────────────────────────────────────┤
│ ┌────────┐ ┌────────┐ ┌────────┐          │
│ │호출    │ │사용자수│ │에러    │          │  ← 요약 카드 3종 (증감률 포함)
│ │14.5K   │ │ 73     │ │ 406    │          │
│ │+12.6%  │ │+15.6%  │ │+19.6%  │          │
│ └────────┘ └────────┘ └────────┘          │
├──────────────────────────────────────────┤
│ 🔍 [검색 입력]                            │
│ ─────────────────────────────────────    │
│ • 로그 1 (시각 / 사용자 / 요약)           │  ← 로그 6~10건
│ • 로그 2                                  │
│ ...                                       │
│ [더 보기 →]                               │  ← 감사 로그 풀뷰로 페이지네이션
└──────────────────────────────────────────┘
```

### 드로어 위치/크기

- **우측 슬라이드 인**: 모달 우측 영역 위에 별도 레이어로 표시 (z-index 모달 + 1)
- **너비**: 480px (모달 가독성 유지를 위해 차트 영역을 직접 점유하지 않음 — 위에 덮음)
- **백드롭**: 살짝 어두움 (드로어 외부 영역 강조), 클릭 시 닫기
- **애니메이션**: 200~300ms 슬라이드 인/아웃 (PDF 16p 명시)

### 닫기 메커니즘

> ⚠️ **ESC 사용 금지** — 모달 마운트 환경 규칙 (CLAUDE.md). ESC는 모달 닫기 UX 계약.

| 방법 | 동작 |
|---|---|
| 헤더 닫기 버튼 (×) | 즉시 닫기 |
| 백드롭 클릭 | 즉시 닫기 |
| 같은 막대 재클릭 | 토글 닫기 (drill-through 패턴 일관) |
| 다른 막대 클릭 | 컨텍스트 전환 (닫지 않음, 새 데이터로 갱신) |
| 메트릭 전환 (drill-through) | 자동 닫기 — 컨텍스트가 모호해지므로 안전을 위해 |

### 헤더

- **부서명 + "팀" 접미사**: `${department_name}팀 활동` 형식. 예: "재무팀 활동", "IT본부팀 활동"
  - 미배정 처리: `department_id == null`일 때는 "미배정 활동" (H-DASH-04 fallback, "팀" 접미사 안 붙임)
- **메트릭 칩**: 활성 메트릭 표시 (예: "API 호출", "오브젝트 분포")
- **CSV 버튼**: 헤더 우측 영역 (닫기 버튼 옆). 화면만, 백엔드 미구현 — 클릭 시 안내 토스트
- **닫기 버튼**: 우상단 `RiCloseLine`, 클릭 시 즉시 닫기

### 컨텍스트 칩

활성 필터 상태를 칩 4종으로 표시 (재정의 [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|drill-through]]·`context-bar`와 일관):

| # | 칩 | 표시 예시 |
|---|---|---|
| 1 | 부서 | "부서: 재무" |
| 2 | 지표 | "지표: 호출" |
| 3 | 기간 | "기간: 7일" |
| 4 | 에러 | "에러: ×" 또는 "에러: ✓" (에러 필터 활성 여부) |

칩은 **표시 전용** (드로어 안에서 칩 클릭으로 필터 변경 불가). 변경은 모달 헤더 `DashboardControls`로만.

### 요약 카드 3종

PDF 16p 명시: **호출 / 사용자 / 에러** 3종. **증감률 포함** (이전 기간 대비).

| 카드 | 표시 | 증감률 | 비고 |
|---|---|---|---|
| 호출 | 호출 수 (K/M 압축) | `+12.6%` 식 | API 호출 KPI 동일 규칙, **사용자 기간 따름** |
| 사용자 수 | 사용자 수 | `+15.6%` 식 | 사용자 KPI 동일 규칙 |
| 에러 | **실패 호출 수 절대값** (예: 406) | `+19.6%` 식 | 에러율 % 아님 — 절대값. 증감률은 이전 기간 대비 |

- 카드 디자인: 기존 KPI 카드와 동일 토큰 (`bg-components-chart-bg`)
- 증감률 색상: 양수 빨강(증가 — 에러는 나쁨), 음수 녹색, 0 회색 (에러 외 카드는 반대 — 양수 녹색)
- 증감률 N/A: 이전 기간 데이터 없거나 분모 0 (H-DASH-05/H-DASH-07)

### 로그 리스트

- **표시**: 6~10건 (PDF 16p)
- **로그 항목 형식**: 시각 / 사용자 / 앱·메트릭 / 요약 (1~2줄)
- **검색**: 상단 입력으로 클라이언트 필터링 (서버 검색은 풀뷰에서)
- **정렬**: 최신순 디폴트
- **빈 상태**: "로그 없음" 안내 (해당 부서 + 메트릭의 데이터 없을 때)

### "더 보기 →" 버튼

- 위치: 로그 리스트 하단
- 클릭 → **감사 로그 풀뷰 페이지로 이동** (현재 컨텍스트 — 부서/메트릭/기간/에러 필터 — 그대로 전달)
- **Phase 2 미구현 영역**: 감사 로그 풀뷰 페이지 자체는 별도 spec 또는 후속 결정 사항. 본 spec에서는 버튼만 노출 + placeholder 토스트로 처리
- ⚠️ "전체 보기" 별도 버튼 없음 — "더 보기"가 곧 풀뷰 페이지네이션 진입점

### CSV 버튼

- **위치**: 헤더 우측 영역 (닫기 버튼 옆)
- 시각만 노출. **화면만, 백엔드 미구현** (SESSION_HISTORY 핵심 결정)
- 클릭 시: 토스트 안내 ("CSV 다운로드는 추후 지원 예정") 또는 disabled 상태 + 툴팁
- 백엔드 호출 코드 작성 금지 (실제 다운로드 미동작)

## 마운트 환경 제약

- **상태 관리**: `drawerContext: DrawerContext | null` React state (URL 사용 불가)
- **소유자**: `DashboardPage` (또는 모달 레벨)
- **모달 닫기 → 드로어 자동 리셋** (state 소멸)
- **ESC 키 미사용** — 모달 닫기 UX 계약 양보 (CLAUDE.md 마운트 환경 규칙)

### 드로어 열림 시 배경 차트 흐림 처리

> 드로어가 모달 내부 영역 위에 슬라이드 인 되므로, 부서별 활동 테이블 등 가려지지 않는 배경 차트가 시각적으로 활성 상태처럼 보일 수 있음 → **흐림(opacity 또는 blur) 처리**로 비활성 표시.

| 상태 | 배경 차트 (예: dept-activity 테이블) |
|---|---|
| 드로어 닫힘 | 정상 (opacity 1, 인터랙션 활성) |
| 드로어 열림 | 흐림 처리 (opacity 0.4 또는 `blur-sm`) + 인터랙션 비활성 (`pointer-events-none`) |

- ⚠️ **화면 크기 조건 확인 필요**: 드로어 너비 480px가 차트 영역을 침범하지 않고 옆에서 보이는 화면 폭에서만 흐림 처리가 의미 있음. 모달 폭이 충분하지 않으면 드로어가 차트를 완전히 덮어 흐림 불필요할 수 있음 — 실 화면 측정 후 결정
- 모달 폭에 따른 동작 결정 후 spec 갱신 필요 (PM/디자이너 확인)

## 키보드 접근성

| 키 | 동작 |
|---|---|
| `Tab` | 닫기 버튼 → CSV 버튼 → 검색 입력 → 로그 항목 → 더 보기 순회 |
| `Enter` | 포커스된 요소 활성화 |
| `Esc` | (드로어 미사용 — 모달 닫기로 양보) |

- 드로어 열림 시 포커스를 닫기 버튼으로 이동 (자연스러운 진입)
- **포커스 트랩 적용**: 드로어 외부로 Tab 이동 차단 (드로어 닫기 전까지 트랩)
- 드로어 닫힘 시 포커스를 트리거 막대로 복귀

## 비즈니스 규칙

### 컨텍스트 일관성

- 트리거 막대의 `department_id`/`department_name` + 활성 메트릭 + 현재 `period`를 컨텍스트로 응답 데이터 fetch
- 드로어 열린 상태에서 메트릭 또는 기간이 변경되면 → **자동 닫기** (컨텍스트 재구성 회피, 사용자 혼란 방지)

### "미배정" 처리

- 트리거 막대가 "미배정" 행이면 `department_id = null` + `department_name = "미배정"` 컨텍스트로 처리
- 백엔드 쿼리에서 `spx_resource_ownership`에 없는 행을 fallback (H-DASH-04)

### 데이터 페칭

- 드로어 열릴 때 `useDrawerData(context)` 훅이 자동 fetch
- queryKey: `['dashboard', 'drawer', department_id, metric, params]`
- staleTime 5분 (전역 일관)
- 같은 컨텍스트 재진입 시 캐시 사용 (재요청 없음)

### 백엔드 미구현 영역

- CSV 다운로드 (위 "CSV 다운로드 버튼" 섹션)
- "전체 보기 →" 풀뷰 (별도 spec 후속)
- → 본 spec 범위에서 백엔드는 **요약 카드 3종 + 로그 6~10건** API만

## Harness 후보 (미검증 — 정식 등록 보류)

> 검증되지 않은 가설은 `defect-catalog.md`에 등록하지 않음. 5단계 테스트 또는 소스 분석 통과 시 정식 ID 부여.

| 후보 코드 | 가설 결함 | 검증 방법 | 출처 강도 |
|---|---|---|---|
| DRAWER-CONTEXT-MISMATCH | 컨텍스트(부서/메트릭) 변경 시 이전 데이터가 잠깐 표시됨 → 사용자가 잘못된 부서 데이터로 오인 | 막대 빠른 연속 클릭 시 응답 순서 검증, 로딩 스켈레톤 노출 여부 | 🟡 중간 — TanStack Query 기본 동작 위에서 실제 발생 가능 |
| DRAWER-CSV-EXPECTATION | CSV 버튼이 동작 안 함을 사용자가 모르고 클릭 → 혼란/불신 | 토스트 안내 또는 disabled 상태 적절성, UX 검증 | 🟡 중간 — 의사결정 항목, 시각 검증 필요 |
| DRAWER-FOCUS-TRAP | 드로어 열려있을 때 Tab이 외부로 빠져 키보드 사용자가 길을 잃음 | 키보드 접근성 테스트 | 🟢 강함 — 일반 다이얼로그 패턴 위반 시 발생 |

> **기 회피 결정 (DRAWER-ESC-PROPAGATION 격)**: 이 패턴은 [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|drill-through]]와 동일하므로 별도 후보로 추가하지 않음. CLAUDE.md "마운트 환경 주의"의 ESC 규칙 적용.

## 기존 컴포넌트 영향

- **`dept-objects` (Phase 1)**: 막대에 `onBarClick(department_id, department_name)` prop 추가. spec 갱신 필요 (별도 작업).
- **`kpi-drill-through` (Phase 2)**: 활성 모드의 부서별 차트들에도 동일 `onBarClick` 추가. spec 갱신 필요.
- **`DashboardPage`**: `useChartDrawer` 훅 호출 + 컨텍스트 보유 + 트리거 차트들에 핸들러 전파.
- **메트릭 전환 시 드로어 자동 닫기**: drill-through의 `useKpiDrillThrough` 훅과 결합 (`activeMetric` 변경 감지 → 드로어 닫기).

## 관련 노트

- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|drill-through requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md|dept-objects requirements]]
- [[2. 회의록/0430 AAI 주간 보고.md|0430 회의록 - PDF 16p 메모]]
