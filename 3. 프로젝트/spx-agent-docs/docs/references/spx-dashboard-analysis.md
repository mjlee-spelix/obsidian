# spx-agent 대시보드 UI 분석

> 2026-05-29 분석. 프론트엔드 코드 기반 (`web/app/components/admin/`).
> 대시보드 문서 작성 시 참조용.
>
> ⚠️ **본 분석본은 화면·차트·API 구조 박제용(구조 나열)입니다. 본문은 이 구조를 그대로 옮기지 마시기 바랍니다.** 각 KPI 카드·차트·컬럼·드릴다운마다 "①무엇을 보여주는가(의미) → ②어떤 질문에 답하나/왜 중요한가 → ③업무에 어떻게 활용하나" 3단으로 서술합니다. → [[conventions#데이터 조회·시각화 챕터 서술 원칙 (대시보드·감사로그·모니터링)]]

## 페이지 경로

- URL: `/dashboard`
- 파일: `web/app/(commonLayout)/dashboard/page.tsx`

## 화면 레이아웃

```
┌──────────────────────────────────────────────────────┐
│ [설명 텍스트]                      [기간 ▼] [새로고침] │  sticky 상단바
├──────────────────────────────────────────────────────┤
│ ┌────────┬────────┬────────┬────────┐               │
│ │총 오브젝트│총 이용 앱│총 앱 호출│인기 호출 앱│               │  KPI 카드 4개
│ └────────┴────────┴────────┴────────┘               │
│ ┌────────────────┬────────────────┐                 │
│ │부서별 오브젝트   │모델별 토큰 사용량│                 │  차트 2열
│ └────────────────┴────────────────┘                 │
│ ┌──────────────────────────────────┐                │
│ │        부서별 활동 테이블          │                │  테이블 전폭
│ └──────────────────────────────────┘                │
│                                                      │
│  [KPI 클릭 시 → 드릴-스루 뷰로 전환]                  │
│ ┌────────────────┬────────────────┐                 │
│ │ 드릴 차트 (좌)  │ 드릴 차트 (우)  │                 │
│ └────────────────┴────────────────┘                 │
│ ┌──────────────────────────────────┐                │
│ │         드릴 테이블                │                │
│ └──────────────────────────────────┘                │
└──────────────────────────────────────────────────────┘
```

## 기간 선택

| 옵션 | 키 | 범위 |
|------|-----|------|
| 오늘 | today | 00:00 ~ 23:59 |
| 지난 7일 | last7days | 7일 전 ~ 오늘 (기본값) |
| 지난 30일 | last30days | 30일 전 ~ 오늘 |
| 지난 90일 | last90days | 90일 전 ~ 오늘 |

- URL 쿼리: `?period=<key>` (기본값이면 생략)
- 훅: `usePeriod()` (`use-period.ts`)

## KPI 카드 4종

| # | 제목 | 설명 | API 필드 | 드릴-스루 키 |
|---|------|------|----------|------------|
| 1 | 총 오브젝트 | 기간 종료 시점 앱·지식·도구 총 수 | `count`, `diff_percent`, `new_count` | `objects` |
| 2 | 총 이용 앱 수 | 기간 내 1회 이상 사용된 앱 수 | `adoption.count` | `users` |
| 3 | 총 앱 호출량 | 전체 앱 호출 횟수 (디버그 제외) | `api_calls.count` | `calls` |
| 4 | 인기 호출 앱 | 가장 많이 호출된 앱 이름 + 호출 수 | `app_stats.items[0]` | `apps` |

증감 배지: ▴ 녹색(증가), ▾ 빨강(감소), N/A(비교 불가)

## 종합 뷰 — 차트/테이블

### 부서별 오브젝트 현황 (좌측)

- 파일: `admin/dept-objects-chart/index.tsx`
- 차트: 가로 스택 막대 (앱 #2E90FA / 지식 #15B79E / 도구 #EF6820)
- Y축: 부서명 (내림차순), 마지막에 "미배정"
- 범례 클릭 필터링 가능
- API: `/dashboard/dept-objects`

### 모델별 토큰 사용량 (우측)

- 파일: `admin/model-tokens-chart/index.tsx`
- 차트: 가로 막대 (기본 모델 #2E90FA / 로컬 모델 #15B79E / 미분류 #9CA3AF)
- 우상단에 합계 표시
- API: `/dashboard/model-tokens`

### 부서별 활동 테이블

- 파일: `admin/dept-activity-table/index.tsx`
- 열: 부서 | 신규 앱 | 신규 지식 | 신규 도구 | 호출 | 토큰 사용량
- 정렬: 한글 가나다순 (`localeCompare 'ko-KR'`), 미배정 마지막
- 수치 압축: 1K, 1M (호버 시 정확한 수치)
- API: `/dashboard/dept-activity`

## 드릴-스루 뷰 4종

URL: `?metric=<objects|users|calls|apps>`, ESC 키로 복귀

### objects (총 오브젝트)

| 위치 | 컴포넌트 | 내용 | API |
|------|---------|------|-----|
| 좌측 | `drill-charts/objects/dept-cumulative.tsx` | 부서별 누적 오브젝트 (막대) | `/dashboard/drill/objects/dept-cumulative` |
| 우측 | `drill-charts/objects/top-owners.tsx` | 소유자 Top 10 (막대) | `/dashboard/drill/objects/top-owners` |
| 하단 | `drill-tables/dept-new-creations-table.tsx` | 부서별 신규 생성 (테이블) | `/dashboard/drill/objects/dept-new-creations` |

### users (총 이용 앱 수)

| 위치 | 컴포넌트 | 내용 | API |
|------|---------|------|-----|
| 좌측 | `drill-charts/users/dept-adopted-apps.tsx` | 부서별 이용 앱 수 (막대) | `/dashboard/drill/users/dept-adopted-apps` |
| 우측 | `drill-charts/users/top-users.tsx` | 이용자 Top 10 (#15B79E) | `/dashboard/drill/users/top-users` |
| 하단 | `drill-tables/dept-users-table.tsx` | 부서별 이용 현황 (테이블) | `/dashboard/drill/users/dept-user-activity` |

### calls (총 앱 호출량)

| 위치 | 컴포넌트 | 내용 | API |
|------|---------|------|-----|
| 좌측 | `drill-charts/calls/dept-call-count.tsx` | 부서별 호출 수 (막대) | `/dashboard/drill/calls/dept-call-count` |
| 우측 | `drill-charts/calls/model-call-share.tsx` | 모델별 호출 점유 (막대) | `/dashboard/drill/calls/model-call-share` |
| 하단 | `drill-tables/dept-call-rps-table.tsx` | 부서별 RPS + 추세 (테이블) | `/dashboard/drill/calls/dept-call-rps` |

### apps (인기 호출 앱)

| 위치 | 컴포넌트 | 내용 | API |
|------|---------|------|-----|
| 좌측 | `drill-charts/apps/app-call-top10.tsx` | 호출 수 Top 10 (막대) | `/dashboard/drill/calls/app-call-top10` |
| 우측 | `drill-charts/apps/top-error-apps.tsx` | 에러 Top 10 (#F97066, 빨강) | `/dashboard/drill/apps/top-error-apps` |
| 하단 | `drill-tables/app-stats-table.tsx` | 앱별 통계 (클릭→앱 상세) | `/dashboard/drill/apps/app-stats` |

## API 엔드포인트 전체

| 엔드포인트 | 파라미터 | 용도 |
|-----------|---------|------|
| `/dashboard/kpi` | start, end | KPI 4종 |
| `/dashboard/dept-objects` | start, end | 부서별 오브젝트 |
| `/dashboard/model-tokens` | start, end | 모델별 토큰 |
| `/dashboard/dept-activity` | start, end | 부서별 활동 |
| `/dashboard/drill/objects/dept-cumulative` | (없음) | 부서별 누적 |
| `/dashboard/drill/objects/top-owners` | (없음) | 소유자 Top 10 |
| `/dashboard/drill/objects/dept-new-creations` | start, end | 부서별 신규 |
| `/dashboard/drill/users/dept-adopted-apps` | start, end | 부서별 이용 앱 |
| `/dashboard/drill/users/top-users` | start, end | 이용자 Top 10 |
| `/dashboard/drill/users/dept-user-activity` | start, end | 부서별 사용자 |
| `/dashboard/drill/calls/dept-call-count` | start, end | 부서별 호출 |
| `/dashboard/drill/calls/model-call-share` | start, end | 모델별 호출 |
| `/dashboard/drill/calls/dept-call-rps` | start, end | 부서별 RPS |
| `/dashboard/drill/calls/app-call-top10` | start, end | 앱 호출 Top 10 |
| `/dashboard/drill/apps/top-error-apps` | start, end | 에러 Top 10 |
| `/dashboard/drill/apps/app-stats` | start, end | 앱별 통계 |

## 색상 토큰

| 용도 | Hex | 사용처 |
|------|-----|--------|
| 앱, 기본 차트 | #2E90FA | 파란색 |
| 지식, 사용자 | #15B79E | 청록색 |
| 도구 | #EF6820 | 주황색 |
| 에러 | #F97066 | 빨간색 |
| 미분류 | #9CA3AF | 회색 |

## 숫자 포맷

- 1,000 미만: 그대로 (123)
- 1,000~999,999: K 단위 (1.2K)
- 1,000,000 이상: M 단위 (1.5M)
- 호버 시 정확한 수치 표시

## 상태 관리

- 기간: URL `?period=<key>` — `usePeriod()` 훅
- 드릴-스루: URL `?metric=<key>` — `useKpiDrillThrough()` 훅
- 캐시: React Query, staleTime 5분, queryKey `['dashboard', ...]`

## 반응형

| 화면 | KPI 그리드 | 차트 그리드 |
|------|-----------|-----------|
| 모바일 | 1열 | 1열 |
| 태블릿 (sm) | 2열 | 1열 |
| 데스크탑 (xl) | 4열 | 2열 |
