---
tags: [프로젝트, dify]
status: 진행중
---
# SPX-Agent 관련 소스 파일 목록

[[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md|프로젝트 메인 노트]] 작업에 관련된 소스 파일 정리.
프로젝트 경로: `C:\Users\Administrator\Projects\spx-agent`

```
┌─────────────────────────────────────────────────────────────────┐
│                        프론트엔드 (web/)                          │
│                                                                  │
│  [URL: /app/[appId]/overview]                                    │
│       page.tsx              ← Next.js 페이지 진입점              │
│          │                                                       │
│       chart-view.tsx        ← 날짜 범위 상태, 차트 배치          │
│          │                                                       │
│       app-chart.tsx         ← ECharts 래퍼, createBizChart()    │
│          │                                                       │
│       use-apps.ts           ← TanStack Query (API 호출 훅)      │
│          │                    useAppDailyConversations(appId)    │
│          │                                                       │
│       tag.ts / store.ts     ← 태그 API, Zustand 상태            │
└──────────────────┬──────────────────────────────────────────────┘
                   │ HTTP GET /apps/{id}/statistics/daily-conversations
                   │         ?start=...&end=...
┌──────────────────▼──────────────────────────────────────────────┐
│                        백엔드 (api/)                              │
│                                                                  │
│  컨트롤러 (라우팅 + 파라미터 파싱)                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ statistic.py          → /apps/{id}/statistics/* (8종)   │    │
│  │ workflow_statistic.py → /apps/{id}/workflow/stats (4종) │    │
│  │ app.py                → /apps, /apps/{id}               │    │
│  │ tags.py               → /tags, /tag-bindings/*          │    │
│  │ workspace.py          → /workspaces                     │    │
│  └──────────────────────────────┬──────────────────────────┘    │
│                                 │                                │
│  서비스 (비즈니스 로직)           │                                │
│  ┌──────────────────────────────▼──────────────────────────┐    │
│  │ app_service.py        → 앱 목록, 필터링                   │    │
│  │ tag_service.py        → 태그 CRUD, 바인딩                 │    │
│  │ workspace_service.py  → 워크스페이스 조회                 │    │
│  │ (* statistic.py는 서비스 없이 직접 SQL!)                  │    │
│  └──────────────────────────────┬──────────────────────────┘    │
│                                 │                                │
│  모델 (DB 테이블 매핑)            │                                │
│  ┌──────────────────────────────▼──────────────────────────┐    │
│  │ account.py  → Tenant(워크스페이스), Account              │    │
│  │ model.py    → App, Message, Conversation, Tag            │    │
│  │ workflow.py → WorkflowRun, WorkflowNodeExecution         │    │
│  │ dataset.py  → Dataset, Document, AppDatasetJoin          │    │
│  └──────────────────────────────┬──────────────────────────┘    │
└──────────────────────────────────┼──────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────┐
│                        PostgreSQL                                │
│                                                                  │
│  tenants ──── apps ──── messages          ← 채팅 통계 원본       │
│    │            │         conversations                          │
│    │            │                                                │
│    │          tag_bindings ── tags        ← 그룹화 기준          │
│    │            │                                                │
│    │          app_dataset_joins ── datasets ── documents         │
│    │                                                             │
│    └──── workflow_runs                    ← 워크플로우 통계 원본  │
│              workflow_node_executions                            │
└─────────────────────────────────────────────────────────────────┘

```

- 토큰 저장 흐름 
```
사용자 메시지 전송
      │
model_manager.py    ← LLM 호출, 응답 usage(토큰수) 수신
      │
      ├─ 채팅앱 ──→ easy_ui_based_generate_task_pipeline.py
      │              └─ 즉시(동기) → messages 테이블 저장
      │
      └─ 워크플로우 ─→ persistence.py
                        └─ Celery 큐에 넣음
                              │
                    workflow_execution_tasks.py  (워커가 비동기 실행)
                              └─ workflow_runs 테이블 저장
```
## 백엔드 (api/)

### 모델 (DB 스키마)

| 파일 | 주요 모델 |
|------|----------|
| `api/models/account.py` | Tenant, TenantAccountJoin, TenantAccountRole, Account |
| `api/models/model.py` | App, Message, Conversation, Tag, TagBinding, TenantCreditPool, MessageAgentThought |
| `api/models/workflow.py` | Workflow, WorkflowRun, WorkflowNodeExecution, WorkflowArchiveLog |
| `api/models/dataset.py` | Dataset, Document, AppDatasetJoin |
| `api/models/enums.py` | AppMode, AppStatus, TagType, WorkflowType 등 Enum 정의 |

### 컨트롤러 (API 엔드포인트)

| 파일 | 엔드포인트 |
|------|----------|
| `api/controllers/console/workspace/workspace.py` | `/workspaces`, `/all-workspaces` |
| `api/controllers/console/workspace/members.py` | `/workspaces/current/members` |
| `api/controllers/console/app/app.py` | `GET /apps`, `GET /apps/{id}` |
| `api/controllers/console/app/statistic.py` | `/apps/{id}/statistics/*` (통계 8종) |
| `api/controllers/console/app/workflow_statistic.py` | `/apps/{id}/workflow/statistics/*` (통계 4종) |
| `api/controllers/console/tag/tags.py` | `/tags`, `/tag-bindings/*` |
| `api/controllers/console/datasets/datasets.py` | `GET /datasets` |
| `api/controllers/inner_api/workspace/workspace.py` | `/enterprise/workspace` (엔터프라이즈 전용) |

### 서비스 (비즈니스 로직)

| 파일 | 역할 |
|------|------|
| `api/services/workspace_service.py` | 워크스페이스 정보 조회 |
| `api/services/app_service.py` | 앱 목록 조회, 필터링, 페이지네이션 |
| `api/services/tag_service.py` | 태그 CRUD, 바인딩, 태그별 대상 조회 |
| `api/services/dataset_service.py` | 데이터셋 목록 조회 |
| `api/services/billing_service.py` | 빌링 정보, 플랜 일괄 조회 |
| `api/services/feature_service.py` | 기능 플래그, 라이선스 모델 |
| `api/services/credit_pool_service.py` | 크레딧 풀 관리 |
| `api/services/ops_service.py` | 트레이싱 설정 관리 |

### 토큰 저장 흐름

| 파일 | 역할 |
|------|------|
| `api/core/model_manager.py` | LLM 호출, usage 수신 |
| `api/core/app/task_pipeline/easy_ui_based_generate_task_pipeline.py` | 채팅 토큰 동기 저장 |
| `api/core/app/apps/advanced_chat/generate_task_pipeline.py` | 고급 챗 토큰 동기 저장 |
| `api/core/app/workflow/layers/persistence.py` | 워크플로우 토큰 집계, Celery 태스크 큐잉 |
| `api/tasks/workflow_execution_tasks.py` | WorkflowRun 비동기 저장 (Celery) |
| `api/tasks/workflow_node_execution_tasks.py` | WorkflowNodeExecution 비동기 저장 (Celery) |
| `api/core/repositories/celery_workflow_execution_repository.py` | Celery 태스크 인큐 |
| `api/core/repositories/celery_workflow_node_execution_repository.py` | Celery 태스크 인큐 |

### 설정/인프라

| 파일 | 역할 |
|------|------|
| `api/configs/deploy/__init__.py` | EDITION 설정 (SELF_HOSTED / CLOUD) |
| `api/configs/feature/__init__.py` | ALLOW_CREATE_WORKSPACE 등 기능 플래그 |
| `api/configs/enterprise/__init__.py` | ENTERPRISE_ENABLED 설정 |
| `api/extensions/ext_celery.py` | Celery Beat 스케줄 정의 (정기 작업 16종) |
| `docker/docker-compose.yaml` | api, worker, worker_beat, redis, postgres 컨테이너 |
| `LICENSE` | 다중 테넌트 운영 제한 명시 |

## 프론트엔드 (web/)

### 통계 UI

| 파일 | 역할 |
|------|------|
| `web/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/page.tsx` | 통계 페이지 진입점 |
| `web/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/chart-view.tsx` | 차트 컨테이너 |
| `web/app/components/app/overview/app-chart.tsx` | ECharts 래퍼 컴포넌트 |
| `web/app/components/app/overview/app-chart-utils.ts` | 차트 설정, 데이터 포맷, 색상 테마 |
| `web/app/(commonLayout)/app/(appDetailLayout)/[appId]/layout-main.tsx` | 앱 사이드바 네비게이션 |

### 데이터 페칭 / 상태

| 파일 | 역할 |
|------|------|
| `web/service/use-apps.ts` | TanStack Query 훅 (통계 API 호출) |
| `web/service/tag.ts` | 태그 API 호출 (CRUD, 바인딩) |
| `web/models/app.ts` | 통계 응답 타입 정의 |
| `web/app/components/app/store.ts` | Zustand 스토어 (앱 상세 상태) |

### 태그 UI

| 파일 | 역할 |
|------|------|
| `web/app/components/base/tag-management/filter.tsx` | 태그 필터 드롭다운 |
| `web/app/components/base/tag-management/selector.tsx` | 태그 바인딩 선택기 |
| `web/app/components/base/tag-management/tag-item-editor.tsx` | 태그 생성/수정 모달 |
| `web/app/components/base/tag-management/store.ts` | 태그 Zustand 스토어 |

## 관련 노트
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 소스 분석 현황.md]]
