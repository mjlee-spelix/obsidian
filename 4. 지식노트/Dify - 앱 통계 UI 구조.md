---
tags: [dify, 개발, AI-Agent]
date: 2026-04-24
---
# Dify - 앱 통계 UI 구조

## 핵심
- 기존 통계 페이지는 **앱별 Overview** (`/app/[appId]/overview`)로, 앱 하나의 일별 통계를 차트로 보여줌
- 차트 라이브러리는 **ECharts** (`echarts-for-react`), 데이터 페칭은 **TanStack Query**, 상태 관리는 **Zustand**
- 워크스페이스 전체나 부서별 대시보드는 없음 → 중앙 관리 대시보드는 설정 모달 탭으로 별도 추가 예정 ([[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md|설정 모달 구조 참고]])

## 상세

### 라우팅 구조

```
web/app/(commonLayout)/
├── apps/                         ← 앱 목록 페이지
├── datasets/                     ← 데이터셋 목록
├── explore/                      ← 앱 탐색
├── tools/                        ← 도구
├── plugins/                      ← 플러그인
└── app/(appDetailLayout)/[appId]/
    ├── configuration/            ← 프롬프트 설정
    ├── workflow/                 ← 워크플로우 빌더
    ├── logs/                     ← 대화 로그
    ├── develop/                  ← API 접근
    └── overview/                 ← 통계 페이지 (여기)
        ├── page.tsx              ← 진입점
        ├── chart-view.tsx        ← 차트 컨테이너
        ├── card-view.tsx         ← API/Web 앱 카드
        └── time-range-picker/    ← 기간 선택기
```

### 차트 구성

**앱 모드별 표시 차트:**

프론트엔드는 `appDetail.mode`를 보고 두 플래그로 분기:
- `isChatApp` = CHAT, AGENT_CHAT, ADVANCED_CHAT (workflow가 아닌 것)
- `isWorkflow` = WORKFLOW

| 차트                    | COMPLETION | CHAT / AGENT_CHAT / ADVANCED_CHAT | WORKFLOW |
| --------------------- | :--------: | :-------------------------------: | :------: |
| 일별 메시지 수              |            |                 ✓                 |          |
| 일별 대화 수               |     ✓      |                 ✓                 |          |
| 일별 사용자 수              |     ✓      |                 ✓                 |          |
| 대화당 평균 메시지            |            |                 ✓                 |          |
| 평균 응답 시간              |     ✓      |                                   |          |
| TPS (초당 토큰)           |     ✓      |                 ✓                 |          |
| 사용자 만족도               |     ✓      |                 ✓                 |          |
| 토큰 비용 (messages)      |     ✓      |                 ✓                 |          |
| 일별 실행 수               |            |                                   |    ✓     |
| 일별 터미널 수              |            |                                   |    ✓     |
| 토큰 비용 (workflow_runs) |            |                                   |    ✓     |
| 평균 사용자 인터랙션           |            |                                   |    ✓     |

COMPLETION은 `isChatApp`이 아니고 `isWorkflow`도 아닌 별도 케이스로, Chat 계열과 표시되는 차트가 다름.
→ 상세 쿼리 내용은 [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]] 참고

**차트 팩토리 패턴:**
모든 차트가 `createBizChartComponent()` 팩토리 함수로 생성됨.
색상 테마: 초록(메시지/대화), 주황(사용자), 파랑(비용)

### 데이터 페칭 패턴

**위치:** `web/service/use-apps.ts`

TanStack Query 훅으로 통일:
```typescript
const useAppStatisticsQuery = <T>(metric: string, appId: string, params?: DateRangeParams) => {
  return useQuery<T>({
    queryKey: ['apps', 'statistics', metric, appId, params],
    queryFn: () => get<T>(`/apps/${appId}/statistics/${metric}`, { params }),
  })
}

// 사용 예시
useAppDailyConversations(appId, { start, end })
useAppTokenCosts(appId, { start, end })
```

기간 파라미터: `{ start: 'YYYY-MM-DD HH:mm', end: 'YYYY-MM-DD HH:mm' }`

### 상태 관리

- **Zustand** — 앱 상세 정보, 사이드바 상태, 모달 상태 (`web/app/components/app/store.ts`)
- **TanStack Query** — API 응답 캐싱, 자동 리페치
- **Context Provider** — 워크스페이스/인증 컨텍스트 (`AppContextProvider`)

### 참고할 컨벤션 (앱 통계 기준)

| 항목     | 기존 패턴                                                      | 참고 파일                |
| ------ | ---------------------------------------------------------- | -------------------- |
| 데이터 페칭 | `web/service/`에 훅 파일 생성, TanStack Query 사용                 | `use-apps.ts`        |
| 차트     | ECharts + `createBizChartComponent()` 또는 직접 `ReactECharts` | `app-chart.tsx`      |
| 차트 설정  | `buildChartOptions()` 유틸 활용                                | `app-chart-utils.ts` |
| 상태 관리  | Zustand 스토어 생성                                             | `store.ts`           |
| 스타일    | Tailwind CSS, 2컬럼 그리드 (`grid-cols-1 xl:grid-cols-2`)       | `chart-view.tsx`     |

## 관련 노트
- [[4. 지식노트/Dify - App·Dataset·Workflow 오브젝트 모델 구조.md]]
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 소스 분석 현황.md]]
