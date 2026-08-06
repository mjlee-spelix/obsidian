---
tags: [프로젝트, dify]
status: 진행중
---
# SPX-Agent 소스 분석 현황

[[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md|프로젝트 메인 노트]]의 코드 분석 추적용.

## 분석 완료

### 워크스페이스 모델 & API
중앙 관리의 기본 단위. 커뮤니티 버전은 단일 워크스페이스 제한이라 이 안에서 모든 걸 해결해야 함.
- [x] `Tenant`, `TenantAccountJoin`, `TenantAccountRole` 모델 구조
- [x] 워크스페이스 CRUD API 엔드포인트 (`/console/api/workspaces/*`)
- [x] `/all-workspaces` 어드민 전용 API 확인
- [x] 커뮤니티 버전 다중 워크스페이스 제한 확인 (`ALLOW_CREATE_WORKSPACE=False`, 라이선스 제한)
→ [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md|지식노트]]

### 통계 API
대시보드에 보여줄 데이터의 출처. 현재 앱 단위로만 동작해서 부서/프로젝트 단위 합산 레이어를 새로 만들어야 함.
- [x] 앱 통계 엔드포인트 8종 (`/apps/{id}/statistics/*`)
- [x] 워크플로우 통계 엔드포인트 4종 (`/apps/{id}/workflow/statistics/*`)
- [x] 기존 집계가 모두 앱 단위(app_id)로만 동작하는 것 확인
→ [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md|지식노트]]

### 토큰/통계 DB 스키마
토큰 통계 집계 쿼리를 직접 짜야 하니까 테이블 구조와 관계를 알아야 함.
- [x] `messages` 테이블 (message_tokens, answer_tokens, total_price 등)
- [x] `workflow_runs` 테이블 (total_tokens, elapsed_time 등)
- [x] `workflow_node_executions` 테이블 (execution_metadata JSON)
- [x] 보조 테이블 (conversations, message_agent_thoughts, workflow_trigger_logs, workflow_archive_logs)
- [x] 테이블 간 관계도
→ [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md|지식노트]]

### 토큰 저장 흐름
데이터가 언제 DB에 반영되는지 알아야 실시간 대시보드의 지연 문제를 감안할 수 있음.
- [x] 토큰 출처 (LLM Provider 응답 usage 필드, tiktoken 폴백)
- [x] 채팅 → 동기 저장 (easy_ui/advanced_chat pipeline에서 즉시 commit)
- [x] 워크플로우 → 비동기 저장 (Celery 태스크로 worker가 commit)
→ [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md|지식노트]]

### Celery Beat 정기 작업
오래된 메시지/워크플로우 로그를 정기 삭제하므로, 대시보드 통계 범위에 영향. 보존 기간 설정 확인 필요.
- [x] worker_beat 역할 (스케줄러)
- [x] 정기 작업 목록 (clean_messages, clean_workflow_runlogs 등)
- [x] 통계 원본 데이터 삭제 가능성 확인 → 보존 기간 설정 확인 필요
→ [[4. 지식노트/Dify - Celery Beat 정기 백그라운드 작업.md|지식노트]]

## 미분석

### 모니터링 대상 오브젝트 모델 ✅
대시보드에서 "이 부서에 앱 몇 개, 데이터셋 몇 개" 같은 현황을 보여주려면 이것들의 DB 구조를 알아야 함.
- [x] `App` 모델 상세 — mode 7종, status, enable_site/enable_api, tags 등
- [x] `Dataset` 모델 — permission, document_count, word_count, app_count, tags 등
- [x] `Workflow` 모델 — app_id(1:1), version(draft/published), graph(캔버스 JSON)
- [x] `Document` 모델 — indexing_status, word_count, enabled/archived
- [x] `AppDatasetJoin` — 앱↔데이터셋 다대다 연결 테이블
→ [[4. 지식노트/Dify - App·Dataset·Workflow 오브젝트 모델 구조.md|지식노트]]

### 앱 그룹핑 메커니즘 ✅
부서/프로젝트별 통계의 전제 조건. 앱을 그룹으로 묶는 구조가 없으면 통계를 합산할 기준이 없음. 권대리님 RBAC과 직접 연결됨.
- [x] Dify에 **Tag + TagBinding 시스템이 이미 존재** — 앱(type="app"), 데이터셋(type="knowledge")에 태그 부여 가능
- [x] Tag API 엔드포인트 확인 (CRUD + 바인딩), 앱/데이터셋 목록에서 tag_ids 필터 지원
- [x] 프론트엔드에 태그 필터 UI, 태그 관리 UI 이미 구현됨
- [x] 한계 확인: 평면 구조(계층 없음), 권한 개념 없음, 통계 집계 연동 없음, Project/Department 모델 없음
- [ ] 권대리님 RBAC 작업에서 부서/프로젝트 그룹핑을 어떻게 잡는지 확인 후 연계 → 화면 설계 공유 후 협의
→ [[4. 지식노트/Dify - App·Dataset·Workflow 오브젝트 모델 구조.md|지식노트]]

### 앱 목록/검색 API ✅
대시보드에서 "이 부서의 앱 목록"을 보여줄 때 기존 API를 확장할지, 새로 만들지 판단하기 위해 필요.
- [x] `GET /apps` — mode, name, tag_ids, is_created_by_me 필터, 페이지네이션, created_at DESC 정렬
- [x] `GET /apps/{app_id}` — 상세 (site, tags, workflow 포함)
- [x] `GET /datasets` — keyword, tag_ids 필터, 페이지네이션
- [x] tag_ids 필터가 이미 동작 → 태그 기반 그룹핑 시 기존 API 그대로 활용 가능
→ [[4. 지식노트/Dify - App·Dataset·Workflow 오브젝트 모델 구조.md|지식노트]]

### 앱 통계 UI 구조 ✅
기존 앱별 통계 UI의 패턴(라우팅, 차트, API 호출)을 파악해서 대시보드 차트/훅 작성 시 참고.
- [x] 라우팅: `/app/[appId]/overview/` — 앱별 통계 페이지
- [x] 차트: **ECharts** (`echarts-for-react`), `createBizChartComponent()` 팩토리 패턴
- [x] 데이터 페칭: **TanStack Query** 훅 (`web/service/use-apps.ts`)
- [x] 상태 관리: **Zustand** 스토어
→ [[4. 지식노트/Dify - 앱 통계 UI 구조.md|지식노트]]

### 새 API 엔드포인트 등록 방법 ✅
새 파일 만들어서 Blueprint/Namespace에 등록하는 과정. 백엔드 첫 작업이 이거.
- [x] `api/controllers/console/__init__.py` 등 라우트 등록 파일 확인
- [x] 새 컨트롤러 파일 추가 → Namespace 등록 → URL 매핑 흐름 파악
- [x] 3개 API 레이어 (Console/Service/Inner) 구조 및 인증 방식 확인
- [x] 주요 데코레이터 (`@setup_required`, `@login_required`, `@get_app_model` 등) 확인
- [x] 참고 패턴별 기존 파일 정리 (DB 쿼리, Service 호출, 외부 HTTP API)
→ [[4. 지식노트/Dify - 새 API 엔드포인트 등록 방법.md|지식노트]]

### 로컬 개발 환경 구성
코드 수정 → 확인 사이클이 빨라야 개발 가능. 핫 리로드 여부, 디버깅 방법 등.
- [ ] Docker Compose로 띄운 상태에서 코드 수정 → 반영 사이클 확인 (핫 리로드? 재시작?)
- [ ] 백엔드/프론트엔드 로컬 직접 실행 가능 여부 (디버깅용)

### tenant 컨텍스트 흐름
모든 쿼리의 시작점. `@login_required` → `current_user` → `current_tenant_id`로 워크스페이스를 특정하는 전체 흐름.
- [ ] `current_user.current_tenant_id`로 워크스페이스 데이터를 필터링하는 패턴 확인
- [ ] 새 API에서 "이 워크스페이스 내 모든 앱" 조회 시 컨텍스트 활용 방법

### 요청/응답 검증 패턴 (Pydantic)
새 API의 request/response 모델을 기존 패턴에 맞춰 정의하려면 필요.
- [ ] 기존 Pydantic 모델 위치 (`api/fields/`, `api/core/schemas/` 등)
- [ ] 에러 응답 형식 (400, 403, 404 포맷)

### 프론트엔드 — 설정 모달 탭 등록 ✅
대시보드를 설정 모달(account-setting)의 새 탭으로 추가하는 구조. 별도 라우트가 아니라 모달 내 탭 전환 방식.
- [x] 설정 모달 구조 파악 (`account-setting/index.tsx`, `constants.ts`)
- [x] 탭 추가 방법: `constants.ts` 상수 추가 + `index.tsx` 메뉴·렌더링 추가
- [x] 모달 상태 관리: nuqs (URL 파라미터) + `useAccountSettingModal()` 훅
- [x] 기존 탭 7종 구조 및 조건부 표시 패턴 확인
→ [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md|지식노트]]

### DB 마이그레이션 (Alembic/Flask-Migrate)
태그만으로 그룹핑이 부족하면 새 테이블이 필요할 수 있음. 권대리님 RBAC 테이블 추가 시에도 같은 흐름.
- [ ] `api/migrations/` 구조 및 기존 마이그레이션 패턴 확인
- [ ] 마이그레이션 생성/적용 명령어

### 테스트 구조
최소한 새 API가 동작하는지 검증할 수단이 필요.
- [ ] `api/tests/` 기존 테스트 프레임워크 확인 (pytest?)
- [ ] Swagger UI 자동 생성 여부 (Flask-RESTx) → 수동 테스트 방법

### admin API 인증 구조 (보류)
- [ ] `@admin_required` 데코레이터 동작 방식
- [ ] admin API key 관리 구조
- [ ] 중앙 관리 API에 적용할 인증/인가 방식 결정

### 빌링/크레딧 시스템 심화 (보류)
- [ ] `BillingService`, `FeatureService` 내부 로직 상세
- [ ] 셀프호스트에서 빌링 시스템이 실제로 동작하는지 (CLOUD 전용인지)
