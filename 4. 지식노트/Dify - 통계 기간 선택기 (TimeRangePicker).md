---
tags: [Dify, 개발, 프론트엔드, UI패턴]
date: 2026-04-30
related_files:
  - web/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/chart-view.tsx
  - web/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/time-range-picker/index.tsx
  - web/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/time-range-picker/range-selector.tsx
  - web/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/long-time-range-picker.tsx
  - web/app/components/app/log/filter.tsx
---
# Dify - 통계 기간 선택기 (TimeRangePicker)

> Dify 모니터링/로그 화면의 기간 선택 드롭다운 옵션 매핑.
> SPX-Agent 대시보드 페이지 헤더에서 동일 옵션 채택용 참고.

## 1. 두 가지 변형

Dify는 **에디션에 따라 다른 picker를 사용**:

| 컴포넌트 | 사용 위치 | 특징 |
|---------|---------|------|
| `TimeRangePicker` | 클라우드 에디션 (모니터링) | 짧은 옵션 3개 + DatePicker (사용자 지정 날짜 범위) |
| `LongTimeRangePicker` | 셀프호스트 (모니터링), 로그 페이지 | 긴 옵션 9개 (단순 SimpleSelect, DatePicker 없음) |

분기는 `chart-view.tsx`의 `IS_CLOUD_EDITION`으로 결정.

> **SPX-Agent는 셀프호스트** → `LongTimeRangePicker`의 9개 옵션이 우리 컨벤션이 됨.

## 2. LongTimeRangePicker 옵션 (셀프호스트)

`web/app/components/app/log/filter.tsx:21~30`

```typescript
export const TIME_PERIOD_MAPPING = {
  1: { value: 0,  name: 'today' },          // 오늘
  2: { value: 7,  name: 'last7days' },      // 지난 7일
  3: { value: 28, name: 'last4weeks' },     // 지난 4주
  4: { value: ~90, name: 'last3months' },   // 지난 3개월
  5: { value: ~365, name: 'last12months' }, // 지난 12개월
  6: { value: dynamic, name: 'monthToDate' },   // 이번 달
  7: { value: dynamic, name: 'quarterToDate' }, // 이번 분기
  8: { value: dynamic, name: 'yearToDate' },    // 올해
  9: { value: -1, name: 'allTime' },        // 전체 기간
}
```

- 기본값: `defaultValue="2"` → **`last7days` (지난 7일)**
- i18n 키: `appLog.filter.period.{name}`
- `value === 0` → 오늘 하루 (start/end 모두 today)
- `value === -1` → 전체 (query 자체를 undefined로 보냄)
- 그 외 → `today.subtract(value, 'day')`

## 3. TimeRangePicker (클라우드, 참고용)

`time-range-picker/range-selector.tsx`

```typescript
const TIME_PERIOD_MAPPING = [
  { value: 0,  name: 'today' },
  { value: 7,  name: 'last7days' },
  { value: 30, name: 'last30days' },
]
```

추가로 `DatePicker`를 옆에 배치 → **사용자 지정 날짜 범위** 가능.
선택 시 트리거 라벨이 `"Apr 1 - Apr 30"` 형식으로 바뀜.

## 4. 컴포넌트 스타일

`range-selector.tsx:47`

```tsx
<div className="flex h-8 cursor-pointer items-center space-x-1.5 rounded-lg
                bg-components-input-bg-normal pl-3 pr-2">
  <div className="system-sm-regular text-components-input-text-filled">{name}</div>
  <RiArrowDownSLine className="size-4 text-text-quaternary" />
</div>
```

- 높이: `h-8` (32px)
- 너비: `w-40` (160px)
- 옵션 패널: `w-[200px]` translate (24px 왼쪽으로 보정)
- 선택 표시: 왼쪽 ✔ (`RiCheckLine`, `text-text-accent`)

## 5. 쿼리 포맷

```typescript
const queryDateFormat = 'YYYY-MM-DD HH:mm'
```

페이지 → API로 보낼 때 `start`, `end` 둘 다 이 포맷.

## 6. SPX-Agent 적용 권장

### 옵션 (셀프호스트 기준)

**A안 (Dify 100% 동일)**: LongTimeRangePicker 9개 + 사용자 지정 없음
- 장점: 컴포넌트 직접 재사용 가능, Dify 사용자에게 익숙
- 단점: 어드민 대시보드 용도엔 옵션 과다

**B안 (Dify + 사용자 지정)**: 9개 + DatePicker 추가
- 장점: 모든 케이스 커버
- 단점: 신규 조합 (Dify에 선례 없음)

**C안 (간소화)**: 핵심 5개 (오늘/7일/30일/3개월/사용자 지정)
- 장점: 깔끔
- 단점: Dify와 살짝 다름

### 컴포넌트 재사용 전략

기존 `LongTimeRangePicker` 그대로 import해서 쓰는 것도 가능:
```tsx
import LongTimeRangePicker from '@/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/long-time-range-picker'
import { TIME_PERIOD_MAPPING } from '@/app/components/app/log/filter'
```

하지만 경로가 깊어서 새 위치(`app/components/admin/dashboard-header/time-range-picker.tsx`)로 복사하는 것도 합리적. **Dify 무수정 원칙**상 import는 OK (수정만 금지).

## 관련 노트

- [[4. 지식노트/Dify - 디자인 토큰과 차트·카드 색상 패턴.md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md]]
