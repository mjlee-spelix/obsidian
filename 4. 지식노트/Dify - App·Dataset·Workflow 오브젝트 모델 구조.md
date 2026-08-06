---
tags: [dify, 개발, AI-Agent]
date: 2026-04-24
---
# Dify - App·Dataset·Workflow 오브젝트 모델 구조

## 핵심
- 대시보드에서 모니터링할 대상은 **App(앱)**, **Dataset(데이터셋)**, **Workflow(워크플로우)** 세 가지
- Dify에는 **Tag + TagBinding** 태그 시스템이 이미 존재하며, 앱(type="app")과 데이터셋(type="knowledge")에 태그를 붙일 수 있음 → 부서/프로젝트 그룹핑에 활용 가능
- 앱과 데이터셋은 `AppDatasetJoin` 중간 테이블로 다대다 연결

## 상세

### 모델 관계도

```
Tag ← TagBinding → App (type="app")
                    ├── Workflow (app.workflow_id, 1:1)
                    ├── AppDatasetJoin → Dataset (다대다)
                    │                     └── Document
                    ├── messages (통계)
                    └── workflow_runs (통계)

Tag ← TagBinding → Dataset (type="knowledge")
```

### App (`apps` 테이블)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `tenant_id` | StringUUID | 워크스페이스 |
| `name` | String(255) | 앱 이름 |
| `description` | LongText | 앱 설명 |
| `mode` | Enum(AppMode) | 앱 모드 |
| `status` | Enum(AppStatus) | 상태 (현재 normal만 정의) |
| `workflow_id` | StringUUID | 워크플로우 연결 (nullable) |
| `enable_site` | Boolean | 웹 사이트 활성화 여부 |
| `enable_api` | Boolean | API 활성화 여부 |
| `api_rpm` / `api_rph` | Integer | API 분당/시간당 요청 제한 |
| `created_by` | StringUUID | 생성자 |
| `created_at` / `updated_at` | DateTime | 생성/수정 시각 |

**AppMode 종류:**
- `completion` — 텍스트 완성
- `chat` — 일반 챗
- `advanced-chat` — 고급 챗
- `agent-chat` — 에이전트 챗
- `workflow` — 워크플로우
- `channel` — 채널
- `rag-pipeline` — RAG 파이프라인

**주요 프로퍼티:** `app.tags` (태그 목록), `app.workflow` (워크플로우), `app.site` (공개 사이트), `app.is_agent` (에이전트 여부)

### Dataset (`datasets` 테이블)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `tenant_id` | StringUUID | 워크스페이스 |
| `name` | String(255) | 데이터셋 이름 |
| `permission` | Enum | only_me / all_team_members / partial_members |
| `data_source_type` | Enum | upload_file / notion_import 등 |
| `indexing_technique` | Enum | high_quality / economy |
| `embedding_model` | String(255) | 임베딩 모델명 |
| `created_by` | StringUUID | 생성자 |
| `created_at` / `updated_at` | DateTime | 생성/수정 시각 |

**주요 프로퍼티:** `dataset.document_count` (문서 수), `dataset.word_count` (단어 수), `dataset.app_count` (연결된 앱 수), `dataset.tags` (태그 목록, type="knowledge")

### Document (`documents` 테이블)

Dataset에 속하는 개별 문서.

| 컬럼                | 타입              | 설명                                                 |
| ----------------- | --------------- | -------------------------------------------------- |
| `id`              | StringUUID (PK) |                                                    |
| `dataset_id`      | StringUUID (FK) | 소속 데이터셋                                            |
| `name`            | String(255)     | 문서 이름                                              |
| `indexing_status` | Enum            | waiting / parsing / indexing / completed / error 등 |
| `word_count`      | Integer         | 단어 수                                               |
| `tokens`          | Integer         | 인덱싱 토큰 수                                           |
| `enabled`         | Boolean         | 활성 여부                                              |
| `archived`        | Boolean         | 아카이브 여부                                            |

### Workflow (`workflows` 테이블)

워크플로우의 **설계도** (실행 기록인 `workflow_runs`과 별개).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | StringUUID (PK) | |
| `tenant_id` | StringUUID | 워크스페이스 |
| `app_id` | StringUUID | 소속 앱 (1:1) |
| `type` | Enum(WorkflowType) | workflow / chat / rag-pipeline |
| `version` | String(255) | "draft" 또는 버전 타임스탬프 |
| `graph` | LongText (JSON) | 워크플로우 캔버스 (노드·엣지 정의) |
| `created_by` | StringUUID | 생성자 |
| `created_at` / `updated_at` | DateTime | 생성/수정 시각 |

### Tag 시스템 (`tags` + `tag_bindings` 테이블)

| 테이블 | 주요 컬럼 | 설명 |
|--------|----------|------|
| `tags` | id, tenant_id, type, name | 태그 정의. type은 `app` 또는 `knowledge` |
| `tag_bindings` | tag_id, target_id | 태그-오브젝트 연결. target_id에 앱 또는 데이터셋 id |

- 앱 태그 조회: `app.tags` → TagBinding에서 target_id = app.id, Tag.type = "app"
- 데이터셋 태그 조회: `dataset.tags` → TagBinding에서 target_id = dataset.id, Tag.type = "knowledge"

**Tag API 엔드포인트:**

| 엔드포인트 | 메서드 | 기능 |
|-----------|--------|------|
| `/tags` | GET | 태그 목록 (type, keyword 필터) |
| `/tags` | POST | 태그 생성 (name, type) |
| `/tags/{id}` | PATCH | 태그 이름 수정 |
| `/tags/{id}` | DELETE | 태그 삭제 (바인딩도 같이 삭제) |
| `/tag-bindings/create` | POST | 태그를 앱/데이터셋에 일괄 연결 (tag_ids, target_id, type) |
| `/tag-bindings/remove` | POST | 태그-앱/데이터셋 연결 해제 |

**앱/데이터셋 목록에서 태그 필터링:**
- `GET /apps?tag_ids=uuid1,uuid2` → 해당 태그가 붙은 앱만 반환 (OR 로직)
- `GET /datasets?tag_ids=uuid1,uuid2` → 동일
- 프론트엔드에 태그 필터 UI(멀티 선택 드롭다운), 태그 CRUD UI 이미 구현됨

**부서/프로젝트 그룹핑 관점에서의 한계:**
- 태그는 **평면 구조** — 계층(부서 > 팀 > 프로젝트) 지원 안 됨
- 태그에 **권한 개념 없음** — 누구나 태그 생성/삭제 가능, "부서" 같은 공식 분류로 쓰기엔 느슨함
- **통계 집계에 태그 연동 없음** — 태그별 토큰 합산 같은 기능 없음
- Project / Department 모델은 존재하지 않음

### 앱 목록/검색 API

**`GET /apps`** — 앱 목록 조회 (`api/controllers/console/app/app.py`, `api/services/app_service.py`)

| 파라미터 | 타입 | 설명 |
|---------|------|------|
| `page` / `limit` | int | 페이지네이션 (기본 1/20) |
| `mode` | string | completion / chat / advanced-chat / workflow / agent-chat / channel / all |
| `name` | string | 이름 검색 (앞 30자, ILIKE 부분 일치) |
| `tag_ids` | list | 태그 ID 목록 (OR 로직) |
| `is_created_by_me` | bool | 내가 만든 앱만 |

정렬: `created_at DESC` (최신순), `is_universal == False` 고정 필터

**`GET /apps/{app_id}`** — 앱 상세 (+ site, deleted_tools, api_base_url)

**`GET /datasets`** — 데이터셋 목록 (page, limit, keyword, tag_ids, include_all)

> [!tip] 태그 필터가 이미 동작함
> `tag_ids` 파라미터로 태그 기반 그룹핑 시 기존 API를 그대로 활용 가능. 통계 합산 API만 새로 만들면 됨.

### 오브젝트 간 연결 테이블

| 테이블 | 관계 | 설명 |
|--------|------|------|
| `app_dataset_joins` | App ↔ Dataset (다대다) | 앱이 어떤 데이터셋을 사용하는지 |
| `tag_bindings` | Tag ↔ App/Dataset (다대다) | 태그 분류 |

## 관련 노트
- [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 소스 분석 현황.md]]
