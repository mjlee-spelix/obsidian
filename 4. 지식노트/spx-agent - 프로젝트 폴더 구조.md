---
tags: [dify, 개발, AI-Agent]
date: 2026-04-27
---
# spx-agent 프로젝트 폴더 구조

## 1레벨 폴더

| 폴더 | 역할 |
|------|------|
| api/ | 백엔드 API 서버 코드 |
| dev/ | 개발 환경 설정/스크립트 |
| docker/ | Docker 관련 설정 |
| docs/ | 문서 |
| e2e/ | End-to-End 테스트 |
| images/ | 이미지 파일 |
| packages/ | 모노레포 공유 패키지 |
| scripts/ | 빌드/배포/유틸리티 스크립트 |
| sdks/ | SDK 코드 |
| web/ | 프론트엔드 웹 애플리케이션 |

> pnpm-workspace.yaml 존재 → pnpm 모노레포 구조

---

## 2레벨 폴더

### api/

| 폴더 | 역할 |
|------|------|
| api/commands/ | CLI 커맨드 처리 |
| api/configs/ | 환경 설정 |
| api/constants/ | 상수 정의 |
| api/context/ | 요청 컨텍스트 |
| api/contexts/ | 컨텍스트 모음 |
| api/controllers/ | HTTP 컨트롤러 |
| api/core/ | 핵심 비즈니스 로직 |
| api/docker/ | API용 Docker 설정 |
| api/enterprise/ | 엔터프라이즈 기능 |
| api/enums/ | Enum 정의 |
| api/events/ | 이벤트 처리 |
| api/extensions/ | 확장 기능 |
| api/factories/ | 팩토리 패턴 |
| api/fields/ | 필드/스키마 정의 |
| api/libs/ | 공통 라이브러리 |
| api/migrations/ | DB 마이그레이션 |
| api/models/ | DB 모델 |
| api/repositories/ | 데이터 접근 레이어 |
| api/schedule/ | 스케줄 작업 |
| api/services/ | 서비스 레이어 |
| api/tasks/ | 비동기 태스크 (Celery 등) |
| api/templates/ | 템플릿 파일 |
| api/tests/ | 테스트 코드 |

> 전형적인 Python 백엔드 (Flask/FastAPI) 구조

---

## 3레벨 폴더

### api/controllers/ 하위

> 누가 호출하느냐로 구분되는 구조

| 폴더 | 의미 |
|------|------|
| `console/` | 관리자 대시보드 API — Dify 웹 UI(브라우저)에서 호출 |
| `inner_api/` | 내부 서비스 간 API — 컨테이너끼리 직접 호출, 외부 미노출 |
| `service_api/` | 외부 개발자용 API — API 키로 접근하는 공개 API |
| `web/` | 엔드유저 웹앱 API — 배포된 앱을 일반 사용자가 쓸 때 호출 |
| `common/` | 공통 컨트롤러 — 여러 영역에서 공유하는 기본 클래스/유틸 |
| `files/` | 파일 업로드·다운로드 API |
| `mcp/` | MCP(Model Context Protocol) API — 외부 AI가 도구로 호출 |
| `trigger/` | 트리거 API — 외부 이벤트로 워크플로우 실행 (웹훅 계열) |

#### api/controllers/console/ 하위

| 폴더 | 주요 파일 |
|------|----------|
| `app/` | `statistic.py`, `workflow_statistic.py`, `app.py` — 앱 관련 API 전체 |
| `workspace/` | `workspace.py`, `members.py` — 워크스페이스·멤버·모델 설정 |
| `tag/` | `tags.py` — 태그 CRUD·바인딩 |
| `datasets/` | `datasets.py` — 데이터셋·문서·세그먼트 |
| `auth/` | 로그인·인증 |
| `billing/` | 플랜·빌링 |
| `explore/` | 앱 탐색 |

#### api/controllers/inner_api/ 하위

| 폴더 | 주요 파일 |
|------|----------|
| `workspace/` | `workspace.py` — 엔터프라이즈 전용 워크스페이스 API |
| `app/` | 내부 앱 API |
| `plugin/` | 내부 플러그인 API |

---

### api/models/ 하위

| 폴더 | 의미 |
|------|------|
| `utils/` | 모델 관련 유틸 함수 |
| (직접 파일) | `account.py`, `model.py`, `workflow.py`, `dataset.py` 등 DB 테이블 매핑 |

---

### api/services/ 하위

| 폴더 | 의미 |
|------|------|
| `auth/` | 로그인·인증 로직 |
| `enterprise/` | 엔터프라이즈 전용 비즈니스 로직 |
| `entities/` | 서비스 레이어 데이터 클래스 |
| `errors/` | 서비스 예외 정의 |
| `plugin/` | 플러그인 관련 서비스 |
| `rag_pipeline/` | RAG 파이프라인 처리 |
| `recommend_app/` | 앱 추천 기능 |
| `retention/` | 데이터 보존·삭제 정책 |
| `tools/` | 도구(Tool) 관련 서비스 |
| `trigger/` | 트리거 서비스 |
| `workflow/` | 워크플로우 실행·스케줄 서비스 |
| `document_indexing_proxy/` | 문서 인덱싱 중계 |

---

### api/tasks/ 하위

| 폴더                        | 의미                     |
| ------------------------- | ---------------------- |
| `annotation/`             | 어노테이션 비동기 처리           |
| `app_generate/`           | 앱 생성 관련 Celery 태스크     |
| `rag_pipeline/`           | RAG 인덱싱 비동기 태스크        |
| `workflow_cfs_scheduler/` | 워크플로우 CFS(공정 스케줄링) 태스크 |

---

### api/core/ 하위

| 폴더 | 의미 |
|------|------|
| `app/` | 앱 실행 파이프라인 전체 (핵심) |
| `agent/` | AI 에이전트 실행 로직 |
| `workflow/` | 워크플로우 엔진 |
| `rag/` | RAG 검색·처리 엔진 |
| `tools/` | Tool 실행·관리 |
| `plugin/` | 플러그인 실행 |
| `mcp/` | MCP 프로토콜 처리 |
| `repositories/` | DB 접근 추상화 (Celery/SQLAlchemy) |
| `prompt/` | 프롬프트 빌드·관리 |
| `memory/` | 대화 메모리 관리 |
| `moderation/` | 콘텐츠 필터링 |
| `llm_generator/` | LLM 응답 생성 |
| `ops/` | 트레이싱·옵저버빌리티 |
| `telemetry/` | 텔레메트리 수집 |
| `entities/` | 코어 데이터 클래스 |
| `schemas/` | 데이터 스키마 정의 |
| `datasource/` | 외부 데이터 소스 연결 |
| `external_data_tool/` | 외부 데이터 도구 |
| `extension/` | 코어 확장 포인트 |
| `callback_handler/` | LLM 콜백 처리 |
| `helper/` | 공통 헬퍼 함수 |
| `base/` | 기본 추상 클래스 |
| `db/` | DB 연결·세션 관리 |
| `logging/` | 로깅 설정 |
| `errors/` | 코어 예외 정의 |
| `trigger/` | 트리거 실행 엔진 |

#### api/core/app/ 하위 (4레벨)

| 폴더 | 의미 |
|------|------|
| `task_pipeline/` | 채팅 실행 파이프라인 (`easy_ui_based_generate_task_pipeline.py` — 토큰 동기 저장) |
| `apps/` | 앱 타입별 실행 로직 (`advanced_chat/generate_task_pipeline.py`) |
| `workflow/` | 워크플로우 실행 (`layers/persistence.py` — 토큰 Celery 큐잉) |
| `app_config/` | 앱 설정 구성 |
| `entities/` | 앱 관련 데이터 클래스 |
| `features/` | 기능별 처리 모듈 |
| `llm/` | LLM 호출 래퍼 |
| `layers/` | 실행 레이어 추상화 |
| `file_access/` | 파일 접근 처리 |

#### api/core/repositories/ 주요 파일

| 파일 | 역할 |
|------|------|
| `celery_workflow_execution_repository.py` | 워크플로우 실행 → Celery 큐 인큐 |
| `celery_workflow_node_execution_repository.py` | 노드 실행 → Celery 큐 인큐 |
| `sqlalchemy_workflow_*_repository.py` | DB 직접 저장 (동기) |

---

### api/configs/ 하위

| 폴더 | 의미 |
|------|------|
| `deploy/` | EDITION 설정 (SELF_HOSTED / CLOUD) |
| `enterprise/` | 엔터프라이즈 기능 활성화 |
| `feature/` | 기능 플래그 (ALLOW_CREATE_WORKSPACE 등) |
| `middleware/` | DB·Redis·스토리지 연결 설정 |
| `observability/` | 모니터링·트레이싱 설정 |
| `packaging/` | 패키징·빌드 설정 |
| `extra/` | 기타 부가 설정 |
| `remote_settings_sources/` | 원격 설정 소스 (Nacos 등) |

---

### api/extensions/ 하위

| 폴더 | 의미 |
|------|------|
| `logstore/` | 로그 저장소 확장 |
| `otel/` | OpenTelemetry 계측 확장 |
| `storage/` | 파일 스토리지 확장 (S3, GCS 등) |

---

## web/ 폴더 구조

### 2레벨 폴더 (전체)

| 폴더 | 의미 |
|------|------|
| `app/` | Next.js 앱 라우터 — 페이지·레이아웃·컴포넌트 전체 |
| `service/` | API 호출 함수 (`use-apps.ts`, `tag.ts` 등) |
| `models/` | API 응답 타입 정의 (`app.ts` 등) |
| `hooks/` | 공통 React 훅 |
| `context/` | 전역 Context |
| `constants/` | 상수 정의 |
| `types/` | 전역 TypeScript 타입 |
| `utils/` | 공통 유틸 함수 |
| `i18n/` | 다국어 번역 파일 |
| `i18n-config/` | 다국어 설정 |
| `themes/` | 테마·스타일 변수 |
| `assets/` | 정적 에셋 (아이콘 등) |
| `public/` | 퍼블릭 정적 파일 |
| `config/` | Next.js·앱 설정 |
| `contract/` | API 계약·스펙 정의 |
| `plugins/` | 프론트엔드 플러그인 |
| `scripts/` | 빌드·유틸 스크립트 |
| `bin/` | 실행 바이너리·스크립트 |
| `docker/` | 프론트엔드용 Docker 설정 |
| `docs/` | 프론트엔드 문서 |
| `next/` | Next.js 캐시·빌드 산출물 |
| `__mocks__/` | Jest 모킹 |
| `__tests__/` | 테스트 코드 |
| `test/` | 테스트 유틸·설정 |

---

### 3레벨 — web/app/ 하위

| 폴더 | 의미 |
|------|------|
| `(commonLayout)/` | 로그인 후 공통 레이아웃 — 앱·데이터셋·탐색 등 주요 페이지 |
| `(shareLayout)/` | 외부 공유 앱 레이아웃 |
| `(humanInputLayout)/` | HITL(사람 입력) 레이아웃 |
| `components/` | 재사용 UI 컴포넌트 전체 |
| `signin/` | 로그인 페이지 |
| `signup/` | 회원가입 페이지 |
| `forgot-password/` | 비밀번호 찾기 |
| `reset-password/` | 비밀번호 재설정 |
| `account/` | 계정 설정 페이지 |
| `activate/` | 계정 활성화 |
| `init/` | 초기 설정 페이지 |
| `install/` | 설치 페이지 |
| `oauth-callback/` | OAuth 콜백 처리 |
| `education-apply/` | 교육 플랜 신청 |
| `styles/` | 전역 스타일 |

---

### 4레벨 — web/app/(commonLayout)/ 하위

| 폴더 | 의미 |
|------|------|
| `app/` | **앱 상세 페이지** — overview(통계), 설정, 워크플로우 등 |
| `apps/` | 앱 목록 페이지 |
| `datasets/` | 데이터셋 페이지 |
| `explore/` | 앱 탐색 페이지 |
| `plugins/` | 플러그인 페이지 |
| `tools/` | 도구 페이지 |
| `education-apply/` | 교육 플랜 신청 페이지 |

---

### 4레벨 — web/app/components/ 하위

| 폴더 | 의미 |
|------|------|
| `app/` | **앱 상세 컴포넌트** — `overview/app-chart.tsx` 등 (통계 차트) |
| `base/` | **공통 UI** — `tag-management/` 포함 (태그 필터·바인딩) |
| `apps/` | 앱 목록 컴포넌트 |
| `app-sidebar/` | 앱 사이드바 네비게이션 |
| `workflow/` | 워크플로우 에디터 |
| `workflow-app/` | 워크플로우 앱 전용 UI |
| `datasets/` | 데이터셋 UI |
| `header/` | 상단 네비게이션 |
| `plugins/` | 플러그인 UI |
| `explore/` | 앱 탐색 UI |
| `tools/` | 도구 UI |
| `billing/` | 빌링·플랜 UI |
| `rag-pipeline/` | RAG 파이프라인 UI |
| `share/` | 공유 앱 UI |
| `signin/` | 로그인 UI 컴포넌트 |
| `provider/` | Context Provider 모음 |
| `custom/` | 커스텀 컴포넌트 |
| `develop/` | 개발·디버그 UI |
| `devtools/` | 개발자 도구 UI |
| `goto-anything/` | 빠른 이동 (spotlight 검색) |

---

## docker/ 폴더 구조

### 주요 파일 (직접)

| 파일 | 의미 |
|------|------|
| `docker-compose.yaml` | 메인 컴포즈 — api, worker, web, redis, postgres 등 전체 |
| `docker-compose.middleware.yaml` | 미들웨어만 별도 구성 (DB, Redis 등) |
| `docker-compose-template.yaml` | 컴포즈 자동 생성용 템플릿 |
| `.env` / `.env.example` | 환경변수 설정 파일 |
| `middleware.env.example` | 미들웨어 전용 환경변수 예시 |
| `dify-env-sync.py/sh` | 환경변수 동기화 스크립트 |
| `generate_docker_compose` | 컴포즈 파일 자동 생성 스크립트 |

### 2레벨 폴더 (서비스별 초기화)

| 폴더 | 의미 |
|------|------|
| `nginx/` | Nginx 리버스 프록시 설정 (`nginx.conf`, `proxy.conf`, https 설정) |
| `keycloak/` | Keycloak SSO 설정 — `dify-realm.json` (realm 초기화) |
| `pgvector/` | PostgreSQL + pgvector 벡터 DB 초기화 |
| `elasticsearch/` | Elasticsearch 초기화 |
| `couchbase-server/` | Couchbase 벡터 DB 초기화 |
| `tidb/` | TiDB 분산 DB 설정 |
| `iris/` | InterSystems IRIS DB 초기화 |
| `ssrf_proxy/` | SSRF 방어 프록시 (Squid) 설정 |
| `certbot/` | SSL 인증서 자동 발급·갱신 |
| `startupscripts/` | 컨테이너 시작 시 초기화 스크립트 |
| `volumes/` | 컨테이너 볼륨 마운트 디렉토리 (app, db, redis, sandbox 등) |

> 벡터 DB가 pgvector, elasticsearch, couchbase, tidb, iris 등 다수 선택 가능 구조
> keycloak으로 SSO 연동 지원, ssrf_proxy로 외부 요청 보안 처리

---

## e2e/ 폴더 구조

> Cucumber 기반 E2E 테스트 프레임워크

### 2레벨 폴더

| 폴더 | 의미 |
|------|------|
| `features/` | 테스트 시나리오 전체 |
| `fixtures/` | 테스트 픽스처 (`auth.ts` — 인증 픽스처) |
| `scripts/` | 테스트 실행 스크립트 |
| `support/` | 웹서버 프로세스 관리 (`web-server.ts`) |

### 3레벨 — e2e/features/ 하위

| 폴더 | 의미 |
|------|------|
| `apps/` | 앱 관련 시나리오 |
| `auth/` | 인증 관련 시나리오 |
| `smoke/` | 스모크 테스트 (최소 동작 확인) |
| `step-definitions/` | Cucumber 스텝 구현 |
| `support/` | 테스트 지원 유틸 |

---

## scripts/ 폴더 구조

| 폴더 | 의미 |
|------|------|
| `stress-test/` | 부하 테스트 — Locust 기반 (`locust.conf`, `sse_benchmark.py`) |

> SSE 스트리밍 성능 벤치마크 포함

---

## sdks/ 폴더 구조

> 외부 개발자용 클라이언트 SDK

| 폴더 | 의미 |
|------|------|
| `nodejs-client/` | Node.js SDK (TypeScript, Vite 빌드) |
| `php-client/` | PHP SDK (`dify-client.php` 단일 파일) |

## 관련 노트
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 관련 소스 파일 목록.md]]
- [[3. 프로젝트/SPX-Agent 소스 분석 현황.md]]
- [[4. 지식노트/Dify - App·Dataset·Workflow 오브젝트 모델 구조.md]]
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md]]
- [[4. 지식노트/Dify - Celery Beat 정기 백그라운드 작업.md]]
