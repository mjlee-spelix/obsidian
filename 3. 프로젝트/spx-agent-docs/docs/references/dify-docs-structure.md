# Dify docs 원본 구조 분석

> Phase 1.1~1.2 산출물. `docs.json` 네비게이션 + `en/` 디렉토리 트리.

## 원본 메타데이터

| 항목 | 값 |
|------|----|
| 리포 | `langgenius/dify-docs` |
| 라이선스 | CC BY 4.0 |
| 기본 브랜치 | `main` |
| 빌드 도구 | Mintlify (`docs.json`) |
| 컨텐츠 형식 | MDX (Markdown + React 컴포넌트) |
| 언어 | en / ja / zh (한국어 없음) |
| 번역 파이프라인 | `tools/translate/` (en → ja/zh 자동) |

## 최상위 디렉토리

```
dify-docs/
├── en/              ← 영어 원본 (소스 언어, 모든 수정 여기서)
├── ja/              ← 일본어 (자동 번역)
├── zh/              ← 중국어 (자동 번역)
├── assets/          ← 정적 자산
├── images/          ← 이미지
├── logo/            ← 로고
├── tools/translate/ ← 번역 파이프라인
├── versions/        ← 버전 관리 디렉토리(?)
├── writing-guides/  ← 작성 가이드 (스타일·포맷·용어집)
├── .claude/skills/  ← Claude Code 작성 보조 skills (Dify가 박아둠)
├── docs.json        ← Mintlify 네비게이션·사이드바 정의 ★
├── style.css        ← 커스텀 스타일
├── .mintignore      ← Mintlify 무시 패턴
├── LICENSE          ← CC BY 4.0
├── README.md        ← 컨트리뷰션 가이드
└── AGENTS.md
```

## GitHub Actions 워크플로 (참고)

```
.github/workflows/
├── backport-clear-assignee.yml      ← 백포트 자동화
├── backport.yml
├── check_external_links.yml         ← 외부 링크 검증
├── check_links.yml                  ← 내부 링크 검증
├── sync_docs_analyze.yml            ← 번역 파이프라인 (5개)
├── sync_docs_cleanup.yml
├── sync_docs_execute.yml
├── sync_docs_on_approval.yml
└── sync_docs_update.yml
```

**중요**: deploy 워크플로 없음 → **Mintlify Cloud가 GitHub push 감지해서 자동 배포** 구조. 사내 자체 호스팅 시 별도 빌드·배포 흐름 필요.

## Phase 1.1 — docs.json 네비게이션 구조 (2026-05-29 완료)

### 최상위 구조

`docs.json` → `navigation.languages[0]` (en, default) → `versions[0]` (Latest) → **4개 dropdown**:

| # | Dropdown | Icon | 포팅 대상 | 비고 |
|---|----------|------|----------|------|
| 1 | **Use Dify** | book-open | **✅ 대상** | 이사님 지시 범위 |
| 2 | Self Host | server | ❌ 제외 | |
| 3 | API Reference | code | ❌ 제외 | OpenAPI spec 5개 |
| 4 | Develop Plugin | code-pull-request | ❌ 제외 | |

### Use Dify 섹션 상세 트리

```
Use Dify (dropdown, icon: book-open)
├── Get Started (3)
│   ├── introduction
│   ├── quick-start
│   └── key-concepts
├── Nodes (24)
│   ├── user-input
│   ├── Trigger (sub-group, icon: bolt-lightning) (4)
│   │   ├── overview
│   │   ├── schedule-trigger
│   │   ├── plugin-trigger
│   │   └── webhook-trigger
│   ├── llm
│   ├── knowledge-retrieval
│   ├── answer
│   ├── output
│   ├── agent
│   ├── question-classifier
│   ├── ifelse
│   ├── human-input
│   ├── iteration
│   ├── loop
│   ├── code
│   ├── template
│   ├── variable-aggregator
│   ├── doc-extractor
│   ├── variable-assigner
│   ├── parameter-extractor
│   ├── http-request
│   ├── list-operator
│   └── tools
├── Build (7)
│   ├── shortcut-key
│   ├── goto-anything
│   ├── orchestrate-node
│   ├── predefined-error-handling-logic
│   ├── mcp
│   ├── version-control
│   └── additional-features
├── Debug (4)
│   ├── step-run
│   ├── variable-inspect
│   ├── history-and-logs
│   └── error-type
├── Publish (9)
│   ├── README
│   ├── Web App (sub-group, icon: globe) (5)
│   │   ├── workflow-webapp
│   │   ├── chatflow-webapp
│   │   ├── web-app-settings
│   │   ├── web-app-access
│   │   └── embedding-in-websites
│   ├── publish-mcp
│   ├── developing-with-apis
│   └── publish-to-marketplace
├── Monitor (10)
│   ├── analysis
│   ├── logs
│   ├── annotation-reply
│   └── Integrations (sub-group, icon: grid-2-plus) (7)
│       ├── integrate-langsmith
│       ├── integrate-langfuse
│       ├── integrate-opik
│       ├── integrate-weave
│       ├── integrate-arize
│       ├── integrate-phoenix
│       └── integrate-aliyun
├── Knowledge (20)
│   ├── readme
│   ├── Create Knowledge (sub-group, icon: square-plus)
│   │   ├── Quick Create
│   │   │   ├── introduction
│   │   │   ├── Import Data
│   │   │   │   ├── readme
│   │   │   │   ├── sync-from-notion
│   │   │   │   └── sync-from-website
│   │   │   ├── chunking-and-cleaning-text
│   │   │   └── setting-indexing-methods
│   │   ├── Create from Knowledge Pipeline (7)
│   │   │   ├── readme
│   │   │   ├── create-knowledge-pipeline
│   │   │   ├── knowledge-pipeline-orchestration
│   │   │   ├── publish-knowledge-pipeline
│   │   │   ├── upload-files
│   │   │   ├── manage-knowledge-base
│   │   │   └── authorize-data-source
│   │   └── Connect to External Knowledge (2)
│   │       ├── connect-external-knowledge-base
│   │       └── external-knowledge-api
│   ├── Manage Knowledge (sub-group, icon: gear) (4)
│   │   ├── maintain-knowledge-documents
│   │   ├── introduction
│   │   ├── metadata
│   │   └── maintain-dataset-via-api
│   ├── test-retrieval
│   ├── integrate-knowledge-within-application
│   └── knowledge-request-rate-limit
├── Workspace (10)
│   ├── readme
│   ├── model-providers
│   ├── plugins
│   ├── app-management
│   ├── team-members-management
│   ├── personal-account-management
│   ├── subscription-management
│   └── API Extension (sub-group, icon: puzzle-piece-simple) (4)
│       ├── api-extension
│       ├── external-data-tool-api-extension
│       ├── moderation-api-extension
│       └── cloudflare-worker
└── Tutorials (15)
    ├── Workflow 101 (10)
    │   ├── lesson-01 ~ lesson-10
    ├── simple-chatbot
    ├── twitter-chatflow
    ├── customer-service-bot
    ├── build-ai-image-generation-app
    └── article-reader
```

**총 페이지 수**: ~102 페이지 (Use Dify 섹션만)

### 제외 섹션 요약

| 섹션 | 페이지 수 | 제외 사유 |
|------|----------|----------|
| Self Host | ~11 | 사내 배포팀 관할, 사용자 매뉴얼 범위 밖 |
| API Reference | OpenAPI 5개 | API 개발자 대상, 별도 결정 |
| Develop Plugin | ~30+ | spx-agent 플러그인 개발 없음 |

## Phase 1.2 — en/use-dify/ 디렉토리 구조 (2026-05-29 완료)

```
en/use-dify/
├── getting-started/     (3 files)
├── nodes/               (20 files + trigger/ 4 files)
├── build/               (7 files)
├── debug/               (4 files)
├── publish/             (3 files + webapp/ 5 files)
├── monitor/             (3 files + integrations/ 7 files)
├── knowledge/           (5 files + create-knowledge/ + manage-knowledge/ + knowledge-pipeline/)
├── workspace/           (7 files + api-extension/ 4 files)
└── tutorials/           (5 files + workflow-101/ 10 files)
```

## writing-guides 분석 (Phase 2 전 필독)

원본 Dify의 작성 가이드 (스타일/포맷/용어집)를 우리 [[../conventions]]에 통합·반영해야 함.

```
writing-guides/
├── formatting-guide.md   ← 포맷 규칙
├── style-guide.md(?)     ← 톤·문체
├── glossary.md(?)        ← 용어집
TODO: 실제 파일 목록 확인 + 우리 conventions에 정합
```

## .claude/skills 분석 (보존 결정 필요)

Dify가 박아둔 Claude Code skills (문서 작성 보조). Junction 충돌 처리 시 보존 여부 결정.

```
TODO: .claude/skills/ 내용 파악 + 우리한테 유용한지 평가
```

## 후속 트리거

- Phase 1.3 [[../scope-mapping]] 작성 시 본 트리 참조
- Phase 2 파일럿 시 writing-guides 반영
- Junction 충돌 처리 시 .claude/skills 보존 결정 박제 → [[../decisions]]
