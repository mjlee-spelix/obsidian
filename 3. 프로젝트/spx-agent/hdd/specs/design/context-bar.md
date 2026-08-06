---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: 컨텍스트 바 (Phase 2 동적 인터랙션)
phase: 2
status: "🔒 보류 (2026-05-22)"
harness: []
date: 2026-05-04
last_updated: 2026-05-26
---

> 🔒 **보류 (2026-05-22)** — chart-drawer 5/13 보류 동일 패턴
>
> `page.tsx`에서 `ContextBar` 렌더링만 비활성. 컴포넌트 구현체(`web/app/components/admin/context-bar/*`) 코드 보존. ESC 해제 동작은 현재 KPI 카드 토글에 통합 (`useKpiDrillThrough` 훅이 ESC 일괄 처리).
> 본 spec 본문은 design intent base 참고용으로 보존. 현재 구현 대상 아님.
> 참조: H-DASH-21 (context-bar 보류 결정), `chart-drawer` (5/13 보류) 동일 패턴. 재도입 가능성 열림 (PM 결정 동반 트리거).

# 컨텍스트 바 — Design

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/context-bar.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/context-bar.md|Tasks]]

## 상태 모델

> 자체 state 없음. `useKpiDrillThrough` 훅의 `activeMetric`을 prop으로 받아 파생 표시 + 해제 콜백 호출만 담당.

```typescript
type KpiMetric = 'objects' | 'users' | 'calls' | 'apps';

interface ContextBarProps {
  activeMetric: KpiMetric | null;
  onDismiss: () => void;       // drill-through `toggle(activeMetric)` 또는 `setActiveMetric(null)`
}
```

## 컴포넌트 구조

```
web/app/components/admin/
└── context-bar/
    ├── index.tsx              ← 컨테이너 + 활성 메트릭 칩 렌더
    └── metric-labels.ts       ← KpiMetric → 한국어 라벨 매핑
```

> 단일 컴포넌트라 폴더 안 만들고 단일 파일(`context-bar.tsx`)로 가도 무방. 향후 칩 종류 확장 시 폴더화.

## 라벨 매핑

```typescript
// metric-labels.ts
export const METRIC_LABELS: Record<KpiMetric, string> = {
  objects: '총 오브젝트',
  users: '사용자 수',
  calls: 'API 호출',
  apps: '앱별 통계',
};
```

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|kpi-cards]] 카드 제목과 일치
- 그레인: `objects/users/calls` → 부서명 단위, `apps` → 앱명 단위 (차트·표가 보여주는 행 단위와 일치)
- 메트릭 추가 도입 시 본 매핑도 갱신

## 컨테이너 (index.tsx)

```typescript
import { RiCloseLine } from '@remixicon/react';
import Badge from '@/app/components/base/badge';
import { cn } from '@/utils/classnames';
import { METRIC_LABELS } from './metric-labels';
import type { KpiMetric } from '@/app/components/admin/kpi-section/types';

interface Props {
  activeMetric: KpiMetric | null;
  onDismiss: () => void;
}

export const ContextBar: FC<Props> = ({ activeMetric, onDismiss }) => {
  if (activeMetric === null) return null;  // 종합 뷰에서 미렌더

  const label = `메트릭: ${METRIC_LABELS[activeMetric]}`;

  return (
    <div className="flex items-center gap-2">
      <button
        type="button"
        onClick={onDismiss}
        className={cn(
          'group inline-flex items-center gap-1 rounded-md',
          'transition-colors hover:bg-state-base-hover',
          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-components-input-border-active',
        )}
        aria-label={`활성 메트릭 해제: ${METRIC_LABELS[activeMetric]}`}
        title="클릭하여 종합 뷰로"
      >
        <Badge uppercase={false}>
          <span className="text-text-secondary">{label}</span>
          <RiCloseLine className="ml-1 h-3 w-3 text-text-tertiary group-hover:text-text-secondary" />
        </Badge>
      </button>
    </div>
  );
};
```

> **기존 컴포넌트 활용** (conventions.md § 4 규칙):
> - `base/badge.tsx` — 칩 시각 (`rounded-[5px] border border-divider-deep`, 다크/라이트 자동)
> - `RiCloseLine` (Remix Icon) — 닫기 아이콘
> - 단, badge 자체에는 클릭 핸들러 없으므로 `<button>`으로 감쌈 (접근성 + 포커스 링)

## DashboardPage 통합

```typescript
// app/(commonLayout)/dashboard/page.tsx (예시)
const drillThrough = useKpiDrillThrough();

return (
  <div className="flex flex-col gap-6">
    <KpiSection
      periodLabel={period.label}
      activeMetric={drillThrough.activeMetric}
      onActivate={drillThrough.toggle}
    />
    <ContextBar
      activeMetric={drillThrough.activeMetric}
      onDismiss={() => drillThrough.setActiveMetric(null)}
    />
    <ChartArea activeMetric={drillThrough.activeMetric} period={period} />
    <DeptActivityTable periodLabel={period.label} />
  </div>
);
```

- 컨텍스트 바는 KPI 섹션과 차트 영역 사이에 위치 (페이지 컨텍스트 영역)
- `activeMetric === null`일 때 컴포넌트가 `null` 반환 → 자리도 안 차지 (gap만 남음)

## 디자인 토큰

| 용도 | 토큰 | 비고 |
|---|---|---|
| 칩 시각 | `base/badge.tsx` 기본값 | `rounded-[5px] border border-divider-deep` |
| 칩 텍스트 | `text-text-secondary` | 본문 톤 |
| 칩 아이콘 | `text-text-tertiary` (hover 시 `text-text-secondary`) | 미세한 강조 |
| 호버 배경 | `bg-state-base-hover` | 클릭 가능 시각 단서 |
| 포커스 링 | `ring-components-input-border-active` | 접근성 |

## API / 백엔드

해당 없음 — 클라이언트 단방향 뷰. 데이터 fetch 없음, queryKey 영향 없음.

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md|drill-through design]]
- [[3. 프로젝트/spx-agent/conventions.md|conventions § 4 기존 토큰/컴포넌트 우선]]
