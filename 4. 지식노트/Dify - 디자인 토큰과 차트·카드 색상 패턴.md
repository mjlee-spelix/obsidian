---
tags: [Dify, 개발, 프론트엔드, 디자인시스템]
date: 2026-04-30
related_files:
  - web/themes/light.css
  - web/themes/dark.css
  - web/themes/tailwind-theme-var-define.ts
  - web/app/components/app/overview/app-chart.tsx
  - web/app/components/app/overview/app-chart-utils.ts
---
# Dify - 디자인 토큰과 차트·카드 색상 패턴

> Dify의 색상 시스템과 통계 카드/차트에서 색상을 어떻게 매핑해 쓰는지 정리.
> SPX-Agent 대시보드 KPI 카드/차트 색상 결정에 참고.

## 1. 디자인 토큰 시스템

### 위치
- `web/themes/light.css` — 라이트 테마 CSS 변수 (814줄)
- `web/themes/dark.css` — 다크 테마
- `web/themes/tailwind-theme-var-define.ts` — Tailwind에 노출하는 토큰 매핑 (816줄)

### 카드/차트 컨테이너 토큰
```css
--color-components-card-bg: #fcfcfd;            /* 카드 배경 (라이트) */
--color-components-card-border: #ffffff;
--color-components-card-bg-alt: #ffffff;        /* 강조 카드 */
--color-components-chart-bg: #ffffff;           /* 차트 컨테이너 배경 */
```

### 시맨틱 텍스트 토큰
```css
--color-text-secondary: #354052;
--color-text-tertiary: #676f83;
--color-text-success: #079455;       /* 증감 상승 */
--color-text-destructive: #d92d20;   /* 증감 하락 */
--color-text-warning: #dc6803;
--color-text-accent: #155aef;        /* 브랜드 강조 */
```

### Util 색상 팔레트 (50~700 단계)

`util-colors-{name}-{name}-{step}` 패턴. 14가지 색상 × 8단계.

| 색상 그룹 | 용도 후보 | 대표값 (500) |
|----------|----------|------------|
| `blue` | API/네트워크 | `#2e90fa` |
| `blue-brand` | 브랜드 강조 | `#296dff` |
| `blue-light` | 부가 정보 | `#0ba5ec` |
| `green` | 성공/활성 | `#17b26a` |
| `teal` | 데이터/지식 | `#15b79e` |
| `orange` | 사용자/알림 | `#ef6820` |
| `orange-dark` | 강한 알림 | `#ff4405` |
| `warning` | 경고 | `#f79009` |
| `yellow` | 노란 강조 | `#eaaa08` |
| `purple` | 자산/오브젝트 | `#7a5af8` |
| `indigo` | 보조 | `#6172f3` |
| `pink` / `fuchsia` | 강조 | `#ee46bc` / `#d444f1` |
| `red` | 위험 | `#f04438` |
| `gray-blue` | 중립 | `#4e5ba6` |

> **사용법**: Tailwind 클래스로 `bg-util-colors-blue-blue-500`, `text-util-colors-green-green-600` 등으로 접근. CSS 변수로는 `var(--color-util-colors-blue-blue-500)`.

## 2. 통계 카드 컨테이너 표준 클래스

`app-chart.tsx:98`에서 확인된 Dify 표준 차트 카드 컨테이너:

```tsx
<div className="flex w-full flex-col rounded-xl bg-components-chart-bg px-6 py-4 shadow-xs">
```

| 속성 | 값 | 의미 |
|------|---|-----|
| 둥글기 | `rounded-xl` | 12px |
| 배경 | `bg-components-chart-bg` | 토큰 사용 (라이트=흰색, 다크=어두운회색) |
| 패딩 | `px-6 py-4` | 24px / 16px |
| 그림자 | `shadow-xs` | 미세한 그림자 |

→ **신규 카드 만들 때 이 클래스를 쓰면 자동으로 라이트/다크 모드 대응.**

## 3. 차트 색상 매핑 — `CHART_TYPE_CONFIG`

`web/app/components/app/overview/app-chart-utils.ts`

### 색상 → 차트 타입 매핑

```typescript
const CHART_TYPE_CONFIG: Record<ChartType, ChartConfig> = {
  messages:      { colorType: 'green' },
  conversations: { colorType: 'green' },
  endUsers:      { colorType: 'orange' },
  costs:         { colorType: 'blue', showTokens: true },
  workflowCosts: { colorType: 'blue' },
}
```

### 색상별 RGBA 값

```typescript
const COLOR_TYPE_MAP = {
  green: {
    lineColor: 'rgba(6, 148, 162, 1)',      // 청록 계열
    bgColor: ['rgba(6, 148, 162, 0.2)', 'rgba(67, 174, 185, 0.08)'],
  },
  orange: {
    lineColor: 'rgba(255, 138, 76, 1)',
    bgColor: ['rgba(254, 145, 87, 0.2)', 'rgba(255, 138, 76, 0.1)'],
  },
  blue: {
    lineColor: 'rgba(28, 100, 242, 1)',
    bgColor: ['rgba(28, 100, 242, 0.3)', 'rgba(28, 100, 242, 0.1)'],
  },
}
```

### 의미적 매핑 패턴 (Dify 컨벤션)

| 영역 | 색상 | 비고 |
|------|------|------|
| 메시지/대화 (활성도) | green | 사용량 증가 = 초록 |
| 사용자 (endUsers) | orange | 사람 관련 |
| 비용/토큰 (costs) | blue | 금액·자원 |

> **주의**: 차트 색은 디자인 토큰이 아닌 raw RGBA로 박혀있다 (echarts 한계). 새 컴포넌트는 가능하면 토큰 기반으로 작성하되, 차트는 토큰 값을 RGBA로 변환해 사용.

## 4. 숫자 포맷 유틸

`@/utils/format`에 `formatNumber` 존재. `app-chart-utils.ts:129`에서 사용:

```typescript
const formattedCost = sumData < 1000
  ? sumData
  : `${formatNumber(Math.round(sumData / 1000))}k`
```

→ 1000 미만은 raw, 이상은 `XXk` 표시. SPX-Agent KPI 카드 숫자 포맷에 재사용 가능.

## 5. 공통 차트 부속 색

```typescript
const COMMON_COLOR_MAP = {
  label: '#9CA3AF',          // 축 라벨
  splitLineLight: '#F3F4F6', // 보조 격자
  splitLineDark: '#E5E7EB',  // 주 격자
}
```

## 6. SPX-Agent 적용 후보 (KPI 카드 4종)

> 결정은 사용자와 상의. 이건 Dify 컨벤션 기반 제안.

### 안 1: Dify 컨벤션 그대로 확장

| KPI 카드 | 색상 | 토큰 | 근거 |
|---------|------|------|------|
| 총 오브젝트 | purple | `util-colors-purple-purple-500` | 자산/누적 — Dify에 없는 새 카테고리 |
| 활성 사용자 | orange | `util-colors-orange-orange-500` | endUsers와 통일 |
| API 호출 | green | `util-colors-green-green-500` | messages와 통일 |
| 토큰 사용 | blue | `util-colors-blue-blue-500` | costs와 통일 |

### 안 2: 시맨틱 단색 (현재 단조 회색 유지 + 강조만 색)

- 카드 배경/제목/숫자 모두 시맨틱 토큰 (text-secondary, text-tertiary)
- 색은 **증감 뱃지에만** (success/destructive)
- 결과: 깔끔, 데이터에 집중. 단점: 카드 간 시각적 구분 약함

### 안 3: 부가 정보(App/KB/Tool)에만 색

- 카드는 단조 회색
- 부가 정보 라벨에만 색 — 부서별 오브젝트 차트의 3색 스택과 통일
- App: `blue-500` (기능)
- KB: `teal-500` 또는 `green-500` (데이터)
- Tool: `orange-500` (확장)

## 관련 노트

- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/dept-objects.md]]
