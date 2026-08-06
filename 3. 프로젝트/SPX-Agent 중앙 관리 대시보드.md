---
tags: [프로젝트, dify, AI-Agent]
stack: [Flask, SQLAlchemy, Next.js, PostgreSQL, Redis]
status: 진행중
---
# SPX-Agent 중앙 관리 대시보드

## 개요
- Dify 셀프호스트(SPX-Agent)에 **워크스페이스 중앙 관리** 기능을 추가하는 프로젝트
- [[2. 회의록/0414 AAI 데일리 스크럼.md|0414 데일리 스크럼]] 관리 기능 7개 중 4번 항목 담당
- 기한: 최대 1달 (4/14 기준)
- ⚠️ 커뮤니티 버전은 **다중 워크스페이스 생성 불가** (엔터프라이즈 전용 + 라이선스 제한) → **단일 워크스페이스 내에서** 부서/프로젝트별 구분 관리하는 방향

## 담당 기능 3가지

### 1. 오브젝트 모니터링 & 중앙 통제
- 여러 부서/업무/프로젝트별 오브젝트(앱, 워크플로우, 데이터셋)를 한눈에 모니터링
- 관리자가 중앙에서 통제할 수 있는 기능 (⚠️ 통제 기능의 범위 정의 필요)

### 2. 통계/모니터링 대시보드
- 기간별 도구 생성 개수, 이용 통계 등
- 부서/프로젝트 단위 사용 현황 집계

### 3. 토큰 사용 통계
- 내부 모델(젬마 등 셀프호스트)과 외부 모델(GPT 등 API) 모두의 토큰 소비량 추적

## 진행 상황
- [x] 담당 기능 범위 확인
- [x] Dify 원본 코드 기존 기능 조사 → [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md|지식노트 참고]]
- [x] Dify 소스 추가 분석 (앱/데이터셋 모델, 프론트엔드 통계 UI 구조, 그룹핑 메커니즘)
- [x] 화면 설계 v0.3 공유 받음 → [[0. Inbox/화면설계_v0.3.pdf]]
- [x] HDD 방법론 적용 → [[3. 프로젝트/SPX-Agent 하네스 설계.md|하네스 설계]] 참고
	- [x] Defect Catalog 작성 → [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|15개 패턴]]
	- [x] HDD 상세 설계 문서 작성 → [[3. 프로젝트/spx-agent/hdd/design.md|design.md]]
	- [x] 화면별 Spec 작성 → 5개 컴포넌트 spec 3종(requirements/design/tasks) 완료
- [x] 김이사님(권대리님 아님) RBAC 스키마 연동 → [[3. 프로젝트/spx-agent/references/rbac-schema.md|rbac-schema.md]]
- [x] 워크플로우 모니터링 데이터 위치 분석 → `workflow_runs` + `workflow_node_executions` (H-DASH-02 처리)
- [x] 백엔드 API 설계 → 5개 spec design 섹션
- [x] 프론트엔드 대시보드 UI 설계 → 5개 spec design 섹션
- [x] **5개 컴포넌트 1단계 프론트 목업 완료** (등급 C 일괄): 대시보드 컨트롤 / KPI 카드 / 부서별 오브젝트 / 모델별 토큰 / 부서별 활동
- [ ] 백엔드 구현 (승랑님 로그/대시보드 스키마 대기 중 — 0430 회의)
- [ ] 설정 모달 사이드바 — 태영님 작업물 수령 후 우리 dashboard 탭 통합
- [ ] KPI 4종 결정 (PDF v0.3는 "에러율 24h", 우리 spec은 "토큰 사용") — 김이사님 협의
- [ ] Phase 2 동적 인터랙션 spec — KPI 클릭 → 차트 변경 / 부서 클릭 → 우측 드로어 (CSV 화면만)
- [ ] "중앙 통제" 기능 — 이번 범위 제외 (비활성/이관/삭제, 토큰 한도, 알림 등)
- [ ] 테스트 (단위 + 통합)

## 주요 이슈/결정사항

| 이슈 | 원인/배경 | 결정 | 관련 노트 |
|------|-----------|------|-----------|
| 커뮤니티 버전 다중 워크스페이스 불가 | `ALLOW_CREATE_WORKSPACE=False` 기본값, 엔터프라이즈 전용 + LICENSE 제한 | 단일 워크스페이스 내 부서/프로젝트별 그룹핑으로 방향 전환 | |
| 통제 기능 정의 | "통제 기능에 대한 정의 및 구현 필요" | **이번 범위 제외** — Phase 2 이후 별도 작업 | [[2. 회의록/0414 AAI 데일리 스크럼.md]] |
| 워크플로우 모니터링 데이터 위치 | 로그 기반인지, DB 기반인지 확인 필요 | **확정** — `workflow_runs.total_tokens` + `workflow_node_executions.process_data` (모델별 분리는 H-DASH-02 미분류 처리) | [[4. 지식노트/Dify - workflow_node_executions에서 모델별 토큰 추출.md]] |
| 통계 원본 데이터 보존 기간 | Celery Beat이 `clean_messages`(매일 4시), `clean_workflow_runlogs`(매일 2시)로 오래된 데이터 삭제 | H-DASH-05로 처리 — 이전 기간 데이터 없으면 증감률 "N/A" 표시 | [[4. 지식노트/Dify - Celery Beat 정기 백그라운드 작업.md]] |
| RBAC 스키마 담당 | 신규: 부서 + 오브젝트 소유 + 권한 | **김이사님** 담당 (권대리님 아님 — 4/29 정정) | |
| RBAC 데이터 저장 전략 | 김이사님 설계 공유됨 | DDL 확정 → `rbac-schema.md` 반영 | [[3. 프로젝트/spx-agent/references/rbac-schema.md]] |
| 부서 구조 (트리 vs 플랫) | `parent_id` 트리 구조 사용 여부 | **플랫 구조 결정** (parent_id 미사용, 각 부서 독립) | |
| 설정 모달 사이드바 충돌 위험 | 본인(대시보드)+김이사님(부서/사용자관리)+승랑님(감사로그)이 같은 컴포넌트 수정 | **태영님이 별도 작업** (0430 회의) → 받아서 우리 dashboard 탭 통합 | [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]] |
| 마운트 환경 | 독립 페이지 vs 설정 모달 탭 | **설정 모달 탭으로 확정** — 모달이 제목 자동 렌더, URL 변경 불가 → React state 기반 | |
| 대시보드 사용자 범위 | 누구에게 보이나 | **관리자(admin) 전용** — URL 북마크/공유 기능 불필요 | |
| KPI 4종 (토큰 사용 vs 에러율 24h) | PDF v0.3는 "에러율", 우리 spec은 "토큰 사용" | **김이사님 협의 필요** — 둘 중 하나 또는 둘 다 | [[2. 회의록/0430 AAI 주간 보고.md]] |
| CSV 내보내기 | 정적 대시보드 vs Phase 2 우측 드로어 | **정적엔 없음**, Phase 2 드로어에서 화면만 (백엔드 미구현) | |
| 로그/대시보드 스키마 | 누가 만드나 | 승랑님 작업 → 받아서 사용 (0430 회의) | [[2. 회의록/0430 AAI 주간 보고.md]] |

## 팀 내 관리 기능 전체 담당 배분

| 기능 | 담당 | 상태 |
|------|------|------|
| SSO 연동 (Keycloak) | 승랑님 | 진행중 |
| RBAC | **김이사님** | DDL 확정 |
| 감사로그 | 승랑님 | 진행중 |
| 로그/대시보드 스키마 | 승랑님 | 진행중 (0430 회의) |
| 설정 모달 사이드바 | 태영님 | 진행중 (0430 회의) |
| **워크스페이스 중앙 관리 (대시보드)** | **본인** | **5개 컴포넌트 1단계 프론트 완료** |
| CI/CD (Jenkins) | 미배정 | 대기 |
| 브랜드/환경 커스터마이징 | 미배정 | 대기 |
| PII 마스킹, 고가용성 | 미배정 | 대기 |

## RBAC 스키마 (확정)

권대리님 쪽에서 만들 테이블. DDL 확인 완료 → [[0. Inbox/테이블추가_설명_DDL_Claude.md]]

| 테이블 | 역할 | 주요 컬럼 | PK |
|--------|------|----------|----|
| `sp_users` | 사용자 계정 (Dify `accounts`와 별도) | `id` UUID, `sub` (OIDC subject, UNIQUE), `email`, `name`, `role` ENUM(admin/user/guest), `is_active` | `id` |
| `sp_departments` | 부서 (트리 구조) | `id` UUID, `code` (Keycloak group path, UNIQUE), `name`, `parent_id` (셀프 참조), `is_active` | `id` |
| `sp_user_departments` | 사용자↔부서 다대다 | `user_id` FK, `department_id` FK, `assigned_at`, `last_seen_at` | `(user_id, department_id)` |
| `sp_object_ownership` | 오브젝트 소속 | `object_type` ENUM(app/dataset/tool), `object_id` UUID, `owner_user_id` FK(변경 불가, 트리거), `department_id` FK | `(object_type, object_id)` |
| `sp_object_permissions` | 오브젝트 접근 권한 | `grantee_type` ENUM(user/department), `grantee_id`, `level` ENUM(viewer/editor), `granted_by` FK | `(object_type, object_id, grantee_type, grantee_id)` |

- Dify 테이블에 FK 없음 (Dify 무수정 원칙) → 정합성은 앱 레이어 + 야간 배치로 보장
- Redis 권한 캐시 (5~15분 TTL)
- JIT 프로비저닝: 첫 로그인 시 `sp_users` INSERT, 이후 로그인 시 email/name UPDATE

### 대시보드에서 이 테이블을 쓰는 방식
- **부서별 오브젝트 분포**: `sp_departments` → `object_ownership.department_id` (직접 연결, object_type별 COUNT)
- **부서별 활동/토큰**: `sp_departments` → `sp_object_ownership` (부서의 앱 목록) → `messages` / `workflow_runs` JOIN
- **부서별 사용자 수**: `sp_departments` → `sp_user_departments` → COUNT
- **부서 구조: 플랫** (`parent_id` 미사용, 각 부서 독립 집계) — 4/29 결정

### 주의할 점
- `sp_object_ownership`은 복합 PK `(object_type, object_id)` → JOIN 시 두 컬럼 모두 사용
- `object_ownership.department_id`로 오브젝트→부서 직접 매핑 (owner 경유 불필요)
- `sp_users` ↔ Dify `accounts` 매핑은 Keycloak upsert 구현 후 확정 (H-DASH-14)
- 부서별 집계 쿼리가 느려질 수 있으므로 Redis 캐시 활용 고려

## 화면 구성별 필요 작업

> 중앙 통제 액션(비활성/이관/삭제, 토큰 한도, 알림 등)은 이번 범위에서 제외. 나중에 별도로 추가.

### KPI 카드 4종 (총 오브젝트 / 활성 사용자 / API 호출 / 토큰 사용)

| KPI | 데이터 출처 | 현재 상태 |
|-----|-----------|----------|
| 총 오브젝트 | `sp_object_ownership` COUNT (type별) | ✅ RBAC 테이블로 가능 |
| 활성 사용자 | `messages.from_end_user_id` + `workflow_runs.created_by` DISTINCT | ✅ 필드 있음, 두 테이블 합산 |
| API 호출 | `messages` COUNT + `workflow_runs` COUNT | ✅ 가능 |
| 토큰 사용 | `messages.(message_tokens + answer_tokens)` + `workflow_runs.total_tokens` SUM | ✅ 가능 |

- 증감(%)은 현재 기간 vs 이전 기간 비교 → 쿼리 2번씩 (이번 30일, 지난 30일)

### 부서별 오브젝트 분포 (3색 스택 막대 차트)
- `sp_departments` → `sp_object_ownership` (object_type별 COUNT)
- `object_type ∈ {app, dataset, tool}` → App/KB/Tool 3색 막대

### 모델별 토큰 사용량 (수평 막대 차트)
- `messages.model_provider`, `model_id`로 GROUP BY → 모델별 토큰 합산
- 화면에 로컬 모델은 **(로컬)** 표시 구분 (예: "Llama 3 (로컬)") → `model_provider`로 로컬/외부 판별 필요
- ⚠️ WORKFLOW 모드는 `workflow_runs`에 모델 정보 없음 → `workflow_node_executions`에서 추출하거나 합산 표시

### 부서별 활동 테이블
- `sp_departments` → `sp_object_ownership` → `apps` → `messages` / `workflow_runs` JOIN
- 컬럼: 신규 App, 신규 KB, 신규 Tool, 호출 수, 토큰 사용 (모두 `+` 부호 prefix, K/M 압축 + hover 정확값)
- 가장 복잡한 쿼리 — 앱 모드에 따라 messages/workflow_runs 테이블 분기 필요 (CTE 5단계로 분리)
- "전체 보기 →" 링크 (우상단) → 별도 상세 페이지 or 모달 (기본은 상위 10개 요약 표시)
- CSV 내보내기는 정적 대시보드에서 제거됨 → Phase 2 우측 드로어에서 화면만 (백엔드 미구현)

### 대시보드 컨트롤 (모달 헤더 우측 슬롯에 삽입)
- 자체 페이지 헤더 만들지 않음 — 모달이 "대시보드" 제목을 자동 렌더
- 컨트롤 구성: 기간 선택 드롭다운 (Dify `LongTimeRangePicker` 9개 옵션 + 사용자 지정 DatePicker) + 새로고침 버튼 (`useIsFetching` 회전)
- 위치: PROVIDER 탭 SearchInput 패턴 따라 모달 헤더 우측 슬롯에 조건부 삽입
- 상태 관리: React state (모달 환경 제약상 URL query string 사용 불가) — `usePeriod` 훅을 `DashboardPage` 또는 모달 레벨이 소유 → prop 전달

### 설정 모달 사이드바 변경 (태영님 작업)
- 기존 '작업 공간' 레이블 제거
- 기존 '멤버' 메뉴 제거
- 메뉴 추가: **부서관리**, **사용자관리**, **대시보드**, **감사로그**
- 0430 회의 결정: **태영님이 별도 작업 진행 중** → 받아서 우리 dashboard 탭 통합 검증 ([[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md|설정 모달 구조 참고]])
- 검증 포인트: 모달 헤더 우측 슬롯에 `DashboardControls` 잘 끼워지는지, 머지 충돌 없는지

## 작업 목록

### 백엔드

| 작업 | 복잡도 | 비고 |
|------|:------:|------|
| API 엔드포인트 등록 (Blueprint) | 하 | 등록 방법 사전 조사 필요 |
| KPI 합산 API | 중 | messages + workflow_runs 양쪽 쿼리 |
| 부서별 오브젝트 수 API | 하 | object_ownership COUNT 쿼리 |
| 모델별 토큰 API | 중 | messages.model_id GROUP BY |
| 부서별 활동 API | 상 | 부서→앱→messages/workflow_runs 다단 JOIN, 앱 모드 분기 |

### 프론트엔드

| 작업 | 상태 | 비고 |
|------|:----:|------|
| 모달 탭 등록 (constants + index) | ✅ | `dashboard-page/` 신규, account-setting 수정 |
| 사이드바 메뉴 추가 | ⏳ 태영님 | 받아서 통합 검증 |
| 대시보드 컨트롤 (기간/새로고침) | ✅ | `dashboard-controls/` (모달 헤더 우측 슬롯) |
| KPI 카드 컴포넌트 | ✅ | 1-A→1-B(디자인 보강)→1-C(모달 통합) |
| 부서별 오브젝트 차트 (수평 스택 막대) | ✅ | ECharts, 합계 라벨, 미배정 막대 |
| 모델별 토큰 차트 (수평 막대) | ✅ | 합계 표시, 로컬 모델 색상 구분 |
| 부서별 활동 테이블 | ✅ | `+`부호, K/M 압축, 미배정 행 |
| TanStack Query 훅 | ⏳ 백엔드 후 | 목업 단계라 미연결 — 스키마 받으면 실데이터 연결 |

## 예상 기간

신입 개발자 기준, 부서 테이블은 만들어진 상태, 기술스택 학습 병행.

| 단계 | 예상 기간 | 내용 |
|------|:---------:|------|
| 사전 준비 | 2~3일 | API 등록 방법, 로컬 개발 환경 구성, 헬로월드 API |
| 백엔드 API | 5~7일 | KPI·부서별·모델별 쿼리 4~5개 + 테스트 |
| 프론트엔드 UI | 5~7일 | 페이지 라우트, 차트 3종, 테이블, 훅 |
| 연동 & 디버깅 | 3~4일 | 백엔드↔프론트 연결, 데이터 확인, 버그 수정 |
| **합계** | **약 3~4주** | 중앙 통제 액션 제외 |

**리스크**:
- ~~부서 테이블 구조가 늦어지면 백엔드 쿼리 착수가 밀림~~ → 로컬에 임시 테이블 만들어서 작업, 나중에 실제 테이블로 교체 (SQLAlchemy 모델 파일만 수정하면 쿼리/서비스 코드는 거의 안 바뀜)
- WORKFLOW 모드 모델별 토큰 분리 불가 — `workflow_runs`에 모델 정보 없음. 합산으로 표시하거나 화면 설계 공유 시 논의 필요
- 부서별 활동 쿼리가 예상보다 복잡할 수 있음 (앱 모드별 테이블 분기)
- 첫 API/프론트 세팅에서 삽질 시간이 예측 어려움

중앙 통제 액션(비활성/이관/삭제, 토큰 한도, 알림)은 별도 1~2주 추가 예상.

## 구현 전략

### 재사용 가능한 기존 코드
- 앱별 통계 API 8종 → 개별 앱 상세 드릴다운
- 토큰·비용 데이터가 Message/WorkflowRun에 이미 저장됨
- 기간 선택기, ECharts 차트 패턴, TanStack Query 훅 패턴

### 신규 개발 필요
- RBAC 테이블(`sp_object_ownership`)과 기존 테이블(`messages`, `workflow_runs`) 연결하는 집계 쿼리
- 부서/모델별 합산 API 엔드포인트
- 중앙 통계 대시보드 페이지 (프론트엔드)
- "중앙 통제" 기능 (앱 비활성화, 권한 제어 등) — 이번 범위 제외

### 아키텍처
```
프론트엔드 (web/app/(commonLayout)/dashboard/)
    ↓ TanStack Query 훅
신규 Admin API 엔드포인트 (api/controllers/console/)
    ↓
DashboardAnalyticsService (신규 서비스)
    ↓
기존 모델 (App, Message, WorkflowRun) + RBAC 모델 (departments, object_ownership)
```

## 기술 스택

| 계층 | 기술 | 관련 내용 |
|------|------|-----------|
| **백엔드 API** | Flask + Flask-RESTx | 통계 API 엔드포인트 (`/apps/{id}/statistics/*`) |
| **ORM / DB** | SQLAlchemy + PostgreSQL | `messages`, `workflow_runs` 등 토큰 데이터 저장 |
| **비동기 작업** | Celery + Redis (Broker) | 워크플로우 토큰 비동기 저장, Beat 정기 작업 |
| **프론트엔드** | Next.js + React + TypeScript | 대시보드 UI 만들 때 여기에 추가 |
| **컨테이너** | Docker Compose | api, worker, worker_beat, redis, postgres 등 |
| **인증** | JWT + Keycloak (OAuth) | 승랑님 SSO 작업과 연결 |

## 관련 노트
- [[2. 회의록/0414 AAI 데일리 스크럼.md]]
- [[4. 지식노트/SPX-Agent - 로컬 개발환경 구성 및 트러블슈팅.md]]
- [[3. 프로젝트/SPX-Agent 소스 분석 현황.md]]
- [[3. 프로젝트/SPX-Agent 관련 소스 파일 목록.md]]
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]
- [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md]]
- [[4. 지식노트/Dify - Celery Beat 정기 백그라운드 작업.md]]
- [[4. 지식노트/Dify - App·Dataset·Workflow 오브젝트 모델 구조.md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]]
- [[4. 지식노트/Dify - 새 API 엔드포인트 등록 방법.md]]
- [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]]
- [[4. 지식노트/spx-agent - 프로젝트 폴더 구조.md]]
- [[4. 지식노트/Dify - workflow_node_executions에서 모델별 토큰 추출.md]]
- [[0. Inbox/Dify - 기술스택 학습가이드.md]]
- [[3. 프로젝트/SPX-Agent 하네스 설계.md|하네스 설계]] — HDD 방법론 적용 계획 + Phase B 실전 검증 결과
- [[3. 프로젝트/spx-agent/CLAUDE.md|CLAUDE.md]] — 에이전트 진입점 지도
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md|SESSION_HISTORY]] — 진행 상태 + 핵심 결정 + 변경 이력
- [[3. 프로젝트/spx-agent/architecture.md|architecture.md]] — 파일 위치 + 등록 방법
- [[3. 프로젝트/spx-agent/conventions.md|conventions.md]] — 코드 스타일 + 색상 매핑
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]] — 도메인 결함 카탈로그 (15개 패턴)
- [[3. 프로젝트/spx-agent/hdd/design.md|HDD 상세 설계]] — 변경 영향 규칙, 데이터 흐름, 스타일 가이드, Harness 매핑
- [[3. 프로젝트/spx-agent/hdd/quality-criteria.md|Quality Criteria]] — 자기 평가 기준
- 컴포넌트별 Spec (5개): [[3. 프로젝트/spx-agent/hdd/specs/requirements/dashboard-controls.md|대시보드 컨트롤]] · [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md|KPI 카드]] · [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md|부서별 오브젝트]] · [[3. 프로젝트/spx-agent/hdd/specs/requirements/model-tokens.md|모델별 토큰]] · [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md|부서별 활동]]
- 회의록: [[2. 회의록/0430 AAI 주간 보고.md|0430 AAI 주간 보고]]
- [[0. Inbox/화면설계_v0.3.pdf|화면설계 v0.3 PDF]]
