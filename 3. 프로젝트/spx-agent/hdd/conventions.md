---
tags: [프로젝트, dify, AI-Agent, HDD]
type: conventions
date: 2026-05-21
last_updated: 2026-06-09
---
# HDD 컨벤션

> 대시보드 UI 영역 공통 표준. 탐색·스튜디오·모니터링 페이지에서 추출한 디자인 토큰 정합 결과.

## UI 디자인 토큰 (5/21 UI 정합 — PPTX 14개 항목 정책화)

### 폰트 표준

| 용도 | 토큰 | 적용 위치 |
|------|------|----------|
| 카드/섹션 제목 | `system-sm-semibold text-text-secondary` | 차트 제목, 표 제목 |
| 부제목/기간 라벨 | `system-2xs-regular text-text-tertiary` | KPI 카드 기간 |
| 표 컬럼 헤더 | `system-xs-medium text-text-tertiary` | 모든 표 `<th>` |
| 표 셀 (텍스트) | `system-sm-medium text-text-secondary` | 부서명, 앱명 |
| 표 셀 (수치) | `system-sm-regular text-text-secondary` | 호출 수, 사용자 수 |
| KPI 큰 숫자 | `system-2xl-semibold text-text-primary` | KPI 카드 주 수치 |
| 세부 라벨 | `system-xs-regular` + 색상 토큰 | App/KB/Tool 분류 |

### 간격 표준

| 용도 | 토큰 | 비고 |
|------|------|------|
| KPI 카드 그리드 간격 | `gap-4` | **변경 금지** — 현행 유지 |
| 차트/표 카드 간격 | `gap-6` | 모니터링 수준 적용 (5/20 1.5) |
| 카드 내부 패딩 | `px-6 py-4` | 전 영역 통일 |
| **카드 하단 패딩** | `pb-5` | 상하 균등화 (5/22 `pb-3` → `pb-5`) |
| 제목→콘텐츠 간격 | `mb-3` | 차트/표 제목 아래 |
| 차트 그리드 (2열) | `grid grid-cols-1 gap-6 xl:grid-cols-2` | 좌하/우하 차트 |
| 표→차트 간격 | `mt-4` | 차트 영역 아래 표 |

### ECharts 여백 표준 (C6 — 5/22 보강)

| 옵션 | 값 | 비고 |
|------|----|----|
| `grid.bottom` | `20` | 전수 (5/22 `10` → `20` — 하단 라벨 잘림 방지) |
| `grid.right` | `60` | 전수 (5/21 박힘 — 우측 수치 라벨 여유) |
| `barWidth` | `16` | 전수 (5/21 박힘 — 가로 막대 두께 통일) |

### 기간 라벨 표시 (C2 — 5/22 신설)

- **적용 대상**: KPI 카드 / 차트 카드 헤더 / 표 카드 헤더 **전수**
- **위치**: 제목 옆 (제목 우측 인라인, `<span className="ml-1.5 ...">`)
- **스타일**: `system-2xs-regular text-text-tertiary` (위 § 폰트 표준 표 재사용)
- **포맷**: 사용자 선택 기간 라벨 그대로 (예: `지난 7 일` / `지난 30 일` / `2026-05-01 ~ 2026-05-22`)
- **i18n**: 기간 라벨 문자열 자체는 `dashboard-controls`의 옵션 라벨(`today` / `last7days` / `last30days` / `last90days` / `custom`)에서 직접 전달
- 카드별 기간 별도 고정 금지 — 모든 영역이 동일 `period` 사용

### 정렬 규칙

- **부서명 기반 표 4개**: 1차가 `localeCompare('ko-KR')` 부서명 오름차순 (지표 무관) — 5/22 코드 정합
  - 대상: `dept-users-table` / `dept-call-rps-table` / `dept-new-creations-table` (그리고 종합 뷰 default `dept-activity-table`)
  - 예외: `app-stats-table`은 기존 규칙(지표 내림차순) 유지 — 부서명 기준이 아니라 앱 그레인
- **기타 표 / 차트**: 1차 주 지표 기준 내림차순 (호출 수 / 오브젝트 수 / 사용자 수 / 에러율 등)
- **2차**: 동률 시 이름 오름차순 (`localeCompare('ko-KR')`) — 한글 ㄱ→ㅎ, 영어 a→z
- **예외**: 미배정/외부 sentinel 행은 항상 마지막

### 긴 텍스트 처리

- Tailwind `truncate` 클래스 (= `overflow-hidden text-ellipsis whitespace-nowrap`)
- hover 시 `title={fullText}` 속성으로 전체 텍스트 노출
- 적용 대상: 차트 막대 라벨 / 표 셀 / Top 앱명 / 부서명 등 전수

### 행 수 고정

| 영역 | 고정 행 수 | 초과 시 |
|------|-----------|--------|
| 부서별 오브젝트 차트 | 6 | 내부 스크롤 |
| 모델별 토큰 차트 | 6 | 내부 스크롤 |
| 부서별 활동 표 | 10 | 내부 스크롤 |

### 빈 상태 / 골격 / 고정 높이 (2026-06-09 — 포인터)

> UI 표준 SoT는 본 문서지만, **빈 상태/골격/결측치/고정높이 정책 전문**은 최상위 `conventions.md § 빈 상태 / 골격 / 결측치 정책`에 있다. 요지:
- 데이터 없어도 **제목/축 골격 유지**(위젯 통째 early-return 금지), 빈 몸통엔 기간 명시 안내 문구
- 결측치 0/없음 → `-`(§ 추세 기호와 일치) / 부서(고정 디멘전) = 전수, 앱·사용자·모델(엔티티/랭킹) = 활성만
- **본문 고정 높이**: 드릴 차트 `h-[250px]` / 표 `h-[392px]` / 모델 토큰 `h-[260px]` / 부서 오브젝트 `h-[280px]` (빈/에러도 동일 높이, `max-h` 금지)

### 추세 기호

| 조건 | 기호 | 색상 |
|------|------|------|
| 양수 | `▴` | `text-text-success` |
| 음수 | `▾` | `text-text-destructive` |
| 0% | 없음 | `text-text-tertiary` |
| 신규 | `+신규` | `text-text-success` |
| 데이터 없음 | `-` | `text-text-tertiary` |

### i18n 라벨

> **금지**: `'App'` / `'KB'` / `'Tool'` 하드코딩. 모든 ECharts series name·legend label·표 컬럼 헤더에서 `t('common.objectType.*')` 사용.

| 키 | 한국어 | 영어 |
|----|--------|------|
| `common.objectType.app` | 앱 | App |
| `common.objectType.kb` | 지식 | KB |
| `common.objectType.tool` | 도구 | Tool |
| `common.menus.dashboardDescription` | 앱, 지식, 도구의 사용 현황과 부서별 활동을 한눈에 확인합니다. | Monitor app, knowledge, and tool usage with department activity at a glance. |

### active 카드 ring

- `ring-inset ring-2 ring-components-button-primary-bg-hover` — 카드 안쪽 ring → 잘림 0 + 레이아웃 무영향
- 비active: `cursor-pointer hover:shadow-lg`

## 색상 체계

| 항목 | 헥스 | 용도 |
|------|------|------|
| App / 기본 | `#2E90FA` (`util-colors-blue-blue-500`) | 앱 카운트, 호출 수 막대 |
| KB / 로컬 모델 | `#15B79E` (`util-colors-teal-teal-500`) | KB 카운트, Top 사용자 막대 |
| Tool | `#EF6820` (`util-colors-orange-orange-500`) | 도구 카운트 |
| 에러 | `#F97066` | 에러 앱 막대 |

## 용어집 (A2 — 5/22 신설)

플랫폼 전역에서 KPI/차트/표 라벨 작성 시 다음 정의를 따른다. 혼용 금지.

| 용어 | 정의 | 사용 위치 |
|------|------|----------|
| **이용** | 앱을 호출/소비하는 활동 (소비자 관점) | KPI 2 `총 이용 앱 수` / 표 `Top 이용 앱` / `부서원당 이용 앱 수` / `이용자` / `앱 이용자 Top10` |
| **사용** | 플랫폼 전반 활동 (생성 + 이용 포함) | 페이지 소개 문구 (`사용 현황과...`) / `모델별 토큰 사용량` / 표 `마지막 사용` |
| **호출** | API 단위 1회 실행 | KPI 3 `총 앱 호출량` / `호출 수 Top10` / `Top 호출 앱` |
| **신규** | 기간 내 생성된 오브젝트 | KPI 1 메인 값 (`new_count`) / 표 `신규` 컬럼 / 증감 뱃지 `+신규` 라벨 |
