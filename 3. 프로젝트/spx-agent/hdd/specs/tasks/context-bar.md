---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: 컨텍스트 바 (Phase 2 동적 인터랙션)
phase: 2
status: "🔒 보류 (2026-05-22)"
harness: []
date: 2026-05-04
last_updated: 2026-05-26
---

> 🔒 **보류 (2026-05-22)** — chart-drawer 5/13 보류 동일 패턴
>
> `page.tsx`에서 `ContextBar` 렌더링만 비활성, 구현체 코드 보존. 본 tasks 체크리스트는 현재 진행 대상 아님 (보류 해제 시 재활성 가능).
> 재도입 결정 시 본 본문 base 활용. 결함 박제: H-DASH-21.

# 컨텍스트 바 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/context-bar.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/context-bar.md|Design]]
> 순서: 컴포넌트 작성 → DashboardPage 통합 → 테스트
> 백엔드 무관 (단방향 뷰)

## 1단계: 컴포넌트 작성

- [ ] 1. `context-bar/metric-labels.ts` — `KpiMetric` → 한국어 라벨 매핑 (`objects` "총 오브젝트" / `users` "사용자 수" / `calls` "API 호출" / `apps` "앱별 통계")
- [ ] 2. `context-bar/index.tsx` — `activeMetric`, `onDismiss` props 받음
- [ ] 3. `activeMetric === null` 시 `null` 반환 (자리도 차지 안 함)
- [ ] 4. `base/badge.tsx` 활용 (기존 컴포넌트 우선 규칙) — `uppercase={false}`로 호출, children에 라벨 + `RiCloseLine`
- [ ] 5. 칩을 `<button>`으로 감싸 클릭 가능하게 + `aria-label` + `title` 툴팁
- [ ] 6. 호버 효과 — `hover:bg-state-base-hover`, 아이콘 색상 강조
- [ ] 7. 포커스 링 — `focus-visible:ring-2 ring-components-input-border-active`

## 2단계: DashboardPage 통합

- [ ] 8. `DashboardPage`에 `useKpiDrillThrough` 훅 호출 (drill-through spec 1단계에서 이미 작업되면 재사용)
- [ ] 9. `<ContextBar>`를 `<KpiSection>`과 `<ChartArea>` 사이(페이지 컨텍스트 영역)에 렌더
- [ ] 10. props 전달 — `activeMetric={drillThrough.activeMetric}`, `onDismiss={() => drillThrough.setActiveMetric(null)}`
- [ ] 11. 시각 검증: KPI 클릭 → 컨텍스트 바 등장, 칩 클릭 → 사라짐

## 3단계: 테스트

- [ ] 12. **CTXBAR-CHIP-DISMISS-LOST 검증**: 칩 × 클릭이 정확히 drill-through state를 null로 만드는지 + KPI 카드의 active 강조도 함께 사라지는지 (단일 진실원 검증)
- [ ] 13. 종합 뷰(`activeMetric === null`)에서 컨텍스트 바가 DOM에 미존재 (height 0이 아니라 아예 없음) 검증
- [ ] 14. 메트릭 전환 시(다른 KPI 클릭) 칩 라벨 즉시 갱신 — 4종 메트릭(`objects/users/calls/apps`) 모두 검증
- [ ] 15. 키보드 접근성: Tab → 칩 포커스 → Enter/Space → 해제
- [ ] 16. ESC 키 동작 확인: 컨텍스트 바가 ESC 가로채지 않음 (drill-through 훅의 ESC 일괄 해제와 충돌 없음)

## 4단계: 검증된 결함 패턴만 defect-catalog 등록

> 검증 안 된 가설은 등록하지 않음.

- [ ] 17. CTXBAR-CHIP-DISMISS-LOST가 실제로 발생 가능한지 또는 단방향 뷰 구조로 원천 차단되는지 판정
- [ ] 18. 발생 가능하면 `H-DASH-XX` 정식 ID 부여 + `hdd/defect-catalog.md` 등록

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/context-bar.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/context-bar.md|Design]]
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]]
