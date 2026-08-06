# 범위 매핑 (3-way)

> Dify 원본 챕터 × spx-agent 유무 × 액션. Phase 1 산출물.
> 작성 시작: 2026-05-29

## 결정 원칙 (5/29 이사님 지시)

- **범위**: Use Dify 섹션만 (Getting Started / Self-host / Plugin Dev / API Reference 등 제외)
- **차감**: spx-agent에 없는 기능 → 삭제
- **추가**: spx-agent 추가 기능 → 신규 챕터
- **언어**: 영어 → 한국어 자연스럽게

## 전역 적용 규칙 (모든 페이지 공통, 2026-06-02 추가)

spx-agent = **Dify Community Edition 1.13.3 기반 fork**. 모든 페이지에서 다음을 일괄 적용:

1. **Dify Cloud/SaaS 플랜 언급 제거** — Free / Pro / Team / Enterprise plan, Subscription, Billing, Seat, Quota 콜아웃 전부 삭제
   - ⚠️ **플래그로 꺼진 유료 기능 주의 (2026-06-15 추가)**: 콜아웃 키워드 없이 "유료/엔터프라이즈로 활성화" 식으로 안내되는 기능도 대상. 예: **모델 제공자 "로드 밸런싱"** — 코드는 존재하나 spx-agent 배포에서 env `MODEL_LB_ENABLED` 기본 `False`로 **비활성**(billing 분기는 SaaS 전용) → 사용자 접근 불가 → 본문 절 삭제. 근거 [[references/spx-workspace-analysis#3.5 Model Providers (`model-providers`)]]. "엔터프라이즈 전용"이 아니라 env 플래그 비활성이므로 배포에서 켜면 복원 가능
2. **"Cloud version에서는..." 분기 콜아웃** → 삭제하거나 CE 기준 단일 서술로 통합
3. **Sandbox / Production 워크스페이스 분리(SaaS 개념)** → CE 단일 환경 기준으로 정리
   - ⚠️ "sandbox" 용어 주의: **SaaS 테스트 워크스페이스**(Cloud의 Sandbox 환경)만 제거 대상. **보안 격리 실행 컨테이너**(`langgenius/dify-sandbox`, Code 노드에서 사용)는 spx-agent에도 실제로 존재하는 컴포넌트 → **보존**
4. **"자체 호스팅 한정" 분기 + 환경 변수/배포 설정 내용**
   - "자체 호스팅 배포의 경우..." 같은 분기 콜아웃은 spx-agent에 항상 참 → 분기 표현 제거
   - 환경 변수·배포 설정 관련 본문(예: `ATTACHMENT_IMAGE_FILE_SIZE_LIMIT` 같은 env var)은 **본문에서 빼고 별도 수집 문서로 추출** → [[references/deployment-config-extracts]] (포팅 중 발견 시 즉시 추가)
   - 추후 수집분 일괄 검토 후 노출 여부 결정 (관리자 가이드 챕터로 묶을지, 폐기할지)
   - "관리자" 표현은 회피 (워크스페이스 관리자 vs 시스템 관리자 모호) — 수집 단계에선 원문 그대로 발췌만
5. **Dify 관련 모든 표기·링크·텍스트 제거 (rebranding 차원)**
   - **Dify 브랜드명·로고 언급** → spx-agent로 교체 또는 통째 제거 ("Dify Workflow" → "워크플로우", "Dify supports..." → "지원합니다" 등)
   - **Dify 공식 외부 채널**: GitHub `langgenius/*`, community / forum / Discord / Twitter / LinkedIn 등 → 즉시 제거
   - **Dify 공식 docs/사이트 링크** (dify.ai, docs.dify.ai, mintlify 브랜딩 등) → 제거
   - **"Powered by Dify" / "Built on Dify" 류 헤더·푸터** → 제거
   - **Marketplace 관련 안내·링크** → **결정 보류** (이사님 컨펌 대기. 일단 원문 유지, 컨펌 결과 따라 일괄 처리)
   - **기능 확장 안내** (예: "사용자 지정 에이전트 전략", "커뮤니티 저장소에서 가져오기" 등) → **본문에서 빼고 별도 수집 문서로 추출** → [[references/feature-extension-extracts]] (Marketplace 결정과 묶어서 일괄 처리 가능)
   - **예외 (보존 대상)**:
     - 루트 `NOTICE.md`의 CC BY 4.0 출처 표기 (라이선스 의무)
     - 내부 코드 컨테이너명·서비스명 (`langgenius/dify-sandbox`, `dify-audit` 등, 운영 식별자라 사용자 노출 X)
6. **Keycloak(KC) 등 외부 인증·관리 시스템 명칭 추상화 (2026-06-04 이사님 결정)**
   - spx-agent 본문에서 KC 같은 외부 시스템 구체 명칭 직접 노출 회피 — 다른 IdP/외부 관리 시스템 사용 환경도 가정
   - **표기 방식**: "외부 시스템(예, Keycloak)" 또는 "관리 시스템" 등 추상 표현으로
   - **로그인/SSO 표현 원칙 (2026-06-05 추가)**: 사용자 입장에서 로그인 방식은 하나다. "SSO로 로그인하는 경우" 같은 조건 분기를 만들지 않는다. spx-agent의 로그인이 SSO를 이용하더라도, 사용자에게는 그냥 "spx-agent에 로그인한다"일 뿐이다. 부서 배정 등 백엔드 동작은 "연동된 계정 관리 시스템에 따라 자동으로" 처리된다고 표현한다.
   - **예시**:
     - "Keycloak에서 멤버 추가합니다" → "외부 시스템에서 멤버 추가합니다" 또는 "관리 시스템에서 멤버를 추가합니다"
     - ~~"KC SSO 로그인" → "SSO 로그인" 또는 "통합 인증 로그인"~~ → "spx-agent에 로그인합니다" (SSO 분기 자체를 만들지 않음)
     - "SSO로 로그인하면 부서가 자동 배정됩니다" → "소속 부서는 연동된 계정 관리 시스템에 따라 자동으로 배정됩니다"
     - "Keycloak 그룹 동기화" → "외부 시스템 그룹 동기화 (예: Keycloak)"
   - 적용 범위: **사용자·부서 관리 신규 챕터에 한정 X — 모든 페이지 본문에서 KC 언급 시 일괄 적용**
   - 사유: spx-agent 솔루션 사용 기업마다 IdP가 다를 수 있음 → 특정 제품 명시는 오해 유발
7. 페이지 액션이 "유지·번역"이어도 위 항목이 본문에 있으면 **번역 시 제거/추출**. 매트릭스 액션은 변경 없음
8. **문장 적합성 필터 (2026-06-09 추가)** — 원본 문장마다 "spx-agent 사용자가 이 안내대로 실행할 수 있는가?" 판단. 제품에 없는 기능을 권고하는 문장, 자체 호스팅 환경과 무관한 운영 맥락은 번역하지 않고 삭제
   - #1~#6은 **키워드**로 잡을 수 있지만, 본 규칙은 키워드 없이 SaaS 맥락이 스며든 문장을 걸러냄 (예: "로그 익명화를 검토하시기 바랍니다" — 제품에 없는 기능 권고)
   - 판단 기준: ① 안내하는 기능/설정이 spx-agent에 실제로 있는가? ② 없다면 해당 문장이 사용자에게 혼란을 주는가? → 없고 혼란을 준다면 삭제
   - ⚠️ **부분 제거 원칙 (2026-06-10 추가)** — 문장을 통째로 지우기 전에, 그 문장에서 SaaS/상업 맥락(요금·플랜·pricing·구독)만 덜어내고 **제품에 실재하는 정보(토큰 소비·동작 제약·기본값 등)는 남겨 재서술**한다. 문장 전체가 SaaS 전용일 때만 통째 삭제.
     - 예: "이 기능은 토큰을 소비합니다. 자세한 내용은 해당 모델의 pricing page를 참고하세요" → "pricing page" 부분만 제거하고 "활성화하면 추가 토큰이 소비됩니다"는 보존
     - 발견 계기: create-knowledge 1차 배치 검토(2026-06-10) — setting-indexing-methods 재순위 모델 3곳에서 "토큰 소비" 유효 정보가 pricing page와 함께 통째 삭제됨
   - 사유: 원본이 SaaS + CE 혼합 독자 대상이라, 번역만으로는 CE 전용 문서에 맞지 않는 문장이 통과됨 (Monitor 리뷰에서 발견)

> 상세 분류 (어떤 게 CE에 포함되고 어떤 게 SaaS 전용인지)는 추후 [[references/dify-edition-comparison]] 작성 (2026-06-02 데일리 노트 deferred 항목).

## Phase 1.1 — Use Dify 섹션 식별 (2026-05-29 완료)

docs.json 분석 결과: **Use Dify** dropdown 1개가 대상. 하위 9개 그룹, ~102 페이지.
상세 트리: [[references/dify-docs-structure]] Phase 1.1 참조.

## Phase 1.3 — 원본 챕터별 액션 매트릭스 (2026-05-29 완료)

> 액션 종류: **유지·번역** / **부분 수정** (번역 + spx 차이 반영) / **삭제** / **검토** (사내 정책 확인 후 결정)

### 1. Get Started (3 pages)

| Dify 챕터      | 경로                           | spx-agent | 액션    | 우선순위 | 비고                                                                      |
| ------------ | ---------------------------- | --------- | ----- | ---- | ----------------------------------------------------------------------- |
| Introduction | getting-started/introduction | ⚠️        | 부분 수정 | P1   | Dify → SPX-Agent 소개로 재작성 (신규 챕터 키워드 티저 포함)                              |
| Quick Start  | getting-started/quick-start  | ⚠️        | 부분 수정 | P1   | 로그인=**사용자명/비밀번호 폼** (KC SSO 아님, 실제 화면 기준), 앱 생성 단계에서 부서 배정·권한 설정 한 줄 언급 |
| Key Concepts | getting-started/key-concepts | ⚠️        | 부분 수정 | P1   | 공통 개념 유지, **부서·RBAC만 추가** (ownership/visibility/ACL은 신규 "권한 설정" 챕터로)    |

### 2. Nodes (24 pages)

| Dify 챕터             | 경로                             | spx-agent | 액션    | 우선순위 | 비고                          |
| ------------------- | ------------------------------ | --------- | ----- | ---- | --------------------------- |
| User Input          | nodes/user-input               | ✅         | 유지·번역 | P2   |                             |
| Trigger Overview    | nodes/trigger/overview         | ✅         | 유지·번역 | P2   |                             |
| Schedule Trigger    | nodes/trigger/schedule-trigger | ✅         | 유지·번역 | P2   |                             |
| Plugin Trigger      | nodes/trigger/plugin-trigger   | ❌         | **삭제** | -    | 이사님 결정(2026-06-04): Marketplace + 플러그인 전체 제거 → Plugin Trigger 자동 비활성 |
| Webhook Trigger     | nodes/trigger/webhook-trigger  | ✅         | 유지·번역 | P2   |                             |
| LLM                 | nodes/llm                      | ✅         | 유지·번역 | P2   |                             |
| Knowledge Retrieval | nodes/knowledge-retrieval      | ✅         | 유지·번역 | P2   |                             |
| Answer              | nodes/answer                   | ✅         | 유지·번역 | P2   |                             |
| Output              | nodes/output                   | ✅         | 유지·번역 | P2   |                             |
| Agent               | nodes/agent                    | ✅         | 유지·번역 | P2   |                             |
| Question Classifier | nodes/question-classifier      | ✅         | 유지·번역 | P2   |                             |
| If/Else             | nodes/ifelse                   | ✅         | 유지·번역 | P2   |                             |
| Human Input         | nodes/human-input              | ✅         | 유지·번역 | P2   |                             |
| Iteration           | nodes/iteration                | ✅         | 유지·번역 | P2   |                             |
| Loop                | nodes/loop                     | ✅         | 유지·번역 | P2   |                             |
| Code                | nodes/code                     | ✅         | 유지·번역 | P2   |                             |
| Template            | nodes/template                 | ✅         | 유지·번역 | P2   |                             |
| Variable Aggregator | nodes/variable-aggregator      | ✅         | 유지·번역 | P2   |                             |
| Doc Extractor       | nodes/doc-extractor            | ✅         | 유지·번역 | P2   |                             |
| Variable Assigner   | nodes/variable-assigner        | ✅         | 유지·번역 | P2   |                             |
| Parameter Extractor | nodes/parameter-extractor      | ✅         | 유지·번역 | P2   |                             |
| HTTP Request        | nodes/http-request             | ✅         | 유지·번역 | P2   |                             |
| List Operator       | nodes/list-operator            | ✅         | 유지·번역 | P2   |                             |
| Tools               | nodes/tools                    | ✅         | 유지·번역 | P2   |                             |

### 3. Build (7 pages)

| Dify 챕터             | 경로                                    | spx-agent | 액션    | 우선순위 | 비고           |
| ------------------- | ------------------------------------- | --------- | ----- | ---- | ------------ |
| Shortcut Key        | build/shortcut-key                    | ✅         | 유지·번역 | P2   |              |
| Goto Anything       | build/goto-anything                   | ✅         | 유지·번역 | P2   |              |
| Orchestrate Node    | build/orchestrate-node                | ✅         | 유지·번역 | P2   |              |
| Error Handling      | build/predefined-error-handling-logic | ✅         | 유지·번역 | P2   |              |
| MCP                 | build/mcp                             | ✅         | 유지·번역 | P2   | 이사님 결정(2026-06-04): MCP 유지 (사내·사외 무관 연동 가능) |
| Version Control     | build/version-control                 | ✅         | 유지·번역 | P2   |              |
| Additional Features | build/additional-features             | ✅         | 유지·번역 | P2   |              |

### 4. Debug (4 pages)

| Dify 챕터 | 경로 | spx-agent | 액션 | 우선순위 | 비고 |
|----------|------|-----------|------|---------|------|
| Step Run | debug/step-run | ✅ | 유지·번역 | P2 | |
| Variable Inspect | debug/variable-inspect | ✅ | 유지·번역 | P2 | |
| History & Logs | debug/history-and-logs | ✅ | 유지·번역 | P2 | |
| Error Type | debug/error-type | ✅ | 유지·번역 | P2 | |

### 5. Publish (9 pages)

| Dify 챕터                | 경로                                   | spx-agent | 액션     | 우선순위 | 비고                                                                                           |
| ---------------------- | ------------------------------------ | --------- | ------ | ---- | -------------------------------------------------------------------------------------------- |
| Publish Overview       | publish/README                       | ✅         | 부분 수정  | P2   | Marketplace 처리는 전역 규칙 #5(보류)에 따라 일괄 결정                                                       |
| Workflow WebApp        | publish/webapp/workflow-webapp       | ✅         | 유지·번역  | P2   |                                                                                              |
| Chatflow WebApp        | publish/webapp/chatflow-webapp       | ✅         | 유지·번역  | P2   |                                                                                              |
| Web App Settings       | publish/webapp/web-app-settings      | ✅         | 유지·번역  | P2   |                                                                                              |
| Web App Access         | publish/webapp/web-app-access        | ❌         | **삭제** | -    | Dify Enterprise 전용 (`tag: "ENTERPRISE"`). spx-agent 미보유. Studio 내 앱 권한은 신규 챕터 "권한 설정"으로 대체 |
| Embedding in Websites  | publish/webapp/embedding-in-websites | ✅         | 유지·번역  | P2   | 이사님 결정(2026-06-04): inbound 영역 유지 (사내 다른 시스템에서 spx-agent WebApp 임베드 가능 가정) |
| Publish as MCP         | publish/publish-mcp                  | ✅         | 유지·번역  | P2   | 이사님 결정(2026-06-04): MCP 유지 (사내·사외 무관 연동 가능)                                                                                 |
| Developing with APIs   | publish/developing-with-apis         | ✅         | 유지·번역  | P2   | 이사님 결정(2026-06-04): inbound 영역 유지 (사내 다른 시스템에서 spx-agent API 호출 가능 가정) |
| Publish to Marketplace | publish/publish-to-marketplace       | ❌         | **삭제** | -    | Marketplace 없음                                                                               |

### 6. Monitor (10 pages)

| Dify 챕터             | 경로                                       | spx-agent | 액션    | 우선순위 | 비고                                                                                              |
| ------------------- | ---------------------------------------- | --------- | ----- | ---- | ----------------------------------------------------------------------------------------------- |
| Analysis            | monitor/analysis                         | ✅         | 부분 수정 | P2   | **제목 "Analysis" → "모니터링"** (spx-agent UI 앱 설정 탭 표기 기준). 신규 챕터 "대시보드"(관리자용)와는 다른 영역(앱별 모니터링)임 명시 |
| Logs                | monitor/logs                             | ✅         | 유지·번역 | P2   |                                                                                                 |
| Annotation Reply    | monitor/annotation-reply                 | ✅         | 유지·번역 | P2   |                                                                                                 |
| Integrate LangSmith | monitor/integrations/integrate-langsmith | ❌         | **삭제** | -    | 이사님 결정(2026-06-04): 외부 SaaS observability 연동 제거                                                                            |
| Integrate Langfuse  | monitor/integrations/integrate-langfuse  | ❌         | **삭제** | -    | 〃                                                                                               |
| Integrate Opik      | monitor/integrations/integrate-opik      | ❌         | **삭제** | -    | 〃                                                                                               |
| Integrate Weave     | monitor/integrations/integrate-weave     | ❌         | **삭제** | -    | 〃                                                                                               |
| Integrate Arize     | monitor/integrations/integrate-arize     | ❌         | **삭제** | -    | 〃                                                                                               |
| Integrate Phoenix   | monitor/integrations/integrate-phoenix   | ❌         | **삭제** | -    | 〃                                                                                               |
| Integrate Aliyun    | monitor/integrations/integrate-aliyun    | ❌         | **삭제** | -    | 〃                                                                                               |

### 7. Knowledge (20 pages)

| Dify 챕터                | 경로                                                            | spx-agent | 액션     | 우선순위 | 비고                                                                                   |
| ---------------------- | ------------------------------------------------------------- | --------- | ------ | ---- | ------------------------------------------------------------------------------------ |
| Knowledge Overview     | knowledge/readme                                              | ✅         | 유지·번역  | P1   |                                                                                      |
| Create Knowledge Intro | knowledge/create-knowledge/introduction                       | ⚠️        | 부분 수정  | P1   | 권한 설정 섹션 추가 + 신규 챕터 "권한 설정" 링크 (Phase 4 진입 전 spx-agent 코드/UI 분석 필요 — 권한 노출 위치·필드 확정) |
| Import Data Overview   | knowledge/create-knowledge/import-text-data/readme            | ✅         | 유지·번역  | P1   |                                                                                      |
| Sync from Notion       | knowledge/create-knowledge/import-text-data/sync-from-notion  | ❌         | **삭제** | -    | 이사님 결정(2026-06-04): 외부 연결 제거 — 외부 데이터 import 폐지                                      |
| Sync from Website      | knowledge/create-knowledge/import-text-data/sync-from-website | ❌         | **삭제** | -    | 〃                                                                                    |
| Chunking & Cleaning    | knowledge/create-knowledge/chunking-and-cleaning-text         | ✅         | 유지·번역  | P1   |                                                                                      |
| Indexing Methods       | knowledge/create-knowledge/setting-indexing-methods           | ✅         | 유지·번역  | P1   |                                                                                      |
| Pipeline Overview      | knowledge/knowledge-pipeline/readme                           | ✅         | 유지·번역  | P2   |                                                                                      |
| Create Pipeline        | knowledge/knowledge-pipeline/create-knowledge-pipeline        | ⚠️        | 부분 수정  | P2   | 파이프라인 내 권한 설정 탭 섹션 추가 + 신규 챕터 "권한 설정" 링크 (Phase 4 진입 전 spx-agent 코드/UI 분석 필요)        |
| Pipeline Orchestration | knowledge/knowledge-pipeline/knowledge-pipeline-orchestration | ✅         | 유지·번역  | P2   |                                                                                      |
| Publish Pipeline       | knowledge/knowledge-pipeline/publish-knowledge-pipeline       | ✅         | 유지·번역  | P2   |                                                                                      |
| Upload Files           | knowledge/knowledge-pipeline/upload-files                     | ✅         | 유지·번역  | P2   |                                                                                      |
| Manage KB              | knowledge/knowledge-pipeline/manage-knowledge-base            | ✅         | 유지·번역  | P2   |                                                                                      |
| Authorize Data Source  | knowledge/knowledge-pipeline/authorize-data-source            | ❌         | **삭제** | -    | 이사님 결정(2026-06-04): 외부 연결 제거                                                         |
| Connect External KB    | knowledge/connect-external-knowledge-base                     | ❌         | **삭제** | -    | 〃                                                                                    |
| External Knowledge API | knowledge/external-knowledge-api                              | ❌         | **삭제** | -    | 〃                                                                                    |
| Maintain Documents     | knowledge/manage-knowledge/maintain-knowledge-documents       | ✅         | 유지·번역  | P1   |                                                                                      |
| Manage KB Settings     | knowledge/manage-knowledge/introduction                       | ⚠️        | 부분 수정  | P1   | 권한 설정 노출 가능성 → 권한 섹션 추가 + 신규 챕터 "권한 설정" 링크 (Phase 4 진입 전 spx-agent 코드/UI 분석 필요)      |
| Metadata               | knowledge/metadata                                            | ✅         | 유지·번역  | P2   |                                                                                      |
| Maintain via API       | knowledge/manage-knowledge/maintain-dataset-via-api           | ✅         | 유지·번역  | P2   | 이사님 결정(2026-06-04): inbound 영역 유지 (사내 다른 시스템에서 spx-agent KB CRUD 가능 가정)              |
| Test Retrieval         | knowledge/test-retrieval                                      | ✅         | 유지·번역  | P1   |                                                                                      |
| Integrate in App       | knowledge/integrate-knowledge-within-application              | ✅         | 유지·번역  | P1   |                                                                                      |
| Rate Limit             | knowledge/knowledge-request-rate-limit                        | ❌         | **삭제** | -    | Dify Cloud 전용 quota. spx-agent 미보유                                                   |

### 8. Workspace (10 pages)

> ⚠️ **재검토 필요 (2026-06-02 표기)**: 본 섹션 전체에 대해 (1) 2026-06-02 신설된 전역 규칙 #1~#5 적용 여부 재점검, (2) spx-agent UI/코드 분석을 통한 비고 정밀화가 필요. 현재 비고는 5/29 분석 시점 가정 기반이라 일부 부정확 가능성 — 예: Personal Account의 "KC SSO 기반 계정" 가정은 실제 로그인이 사용자명/비번 폼인 점을 반영 못 함. 데일리 노트 deferred 항목 참조.

| Dify 챕터              | 경로                                                       | spx-agent | 액션     | 우선순위 | 비고                               |
| -------------------- | -------------------------------------------------------- | --------- | ------ | ---- | -------------------------------- |
| Workspace Overview   | workspace/readme                                         | ⚠️        | 부분 수정  | P1   | spx-agent workspace 구조 반영        |
| Model Providers      | workspace/model-providers                                | ✅         | 유지·번역  | P2   |                                  |
| Plugins              | workspace/plugins                                        | ❌         | **삭제** | -    | Marketplace 없음                   |
| App Management       | workspace/app-management                                 | ✅         | 유지·번역  | P1   |                                  |
| Team Members         | workspace/team-members-management                        | 🔄         | **변환** | P1   | **신규 챕터 "사용자·부서 관리"로 대체**. KC는 본문에 "외부 시스템(예, 키클록)"으로 추상화 ("관리는 외부 시스템에서 수행") |
| Personal Account     | workspace/personal-account-management                    | ⚠️        | 부분 수정  | P2   | KC SSO 기반 계정 → 프로필 설정만           |
| Subscription Mgmt    | workspace/subscription-management                        | ❌         | **삭제** | -    | SaaS 요금제 없음                      |
| API Extension        | workspace/api-extension/api-extension                    | ❌         | **삭제** | -    | 이사님 결정(2026-06-04): outbound 외부 endpoint 호출 — "외부 연결 제거" 적용 |
| External Data Tool   | workspace/api-extension/external-data-tool-api-extension | ❌         | **삭제** | -    | 〃 (워크플로에서 외부 endpoint로 데이터 fetch) |
| Moderation Extension | workspace/api-extension/moderation-api-extension         | ❌         | **삭제** | -    | 〃 (콘텐츠 검열 외부 endpoint 송신) |
| Cloudflare Worker    | workspace/api-extension/cloudflare-worker                | ❌         | **삭제** | -    | SaaS 전용                          |

### 9. Tutorials (15 pages)

| Dify 챕터              | 경로                                      | spx-agent | 액션    | 우선순위 | 비고                                                       |
| -------------------- | --------------------------------------- | --------- | ----- | ---- | -------------------------------------------------------- |
| Workflow 101 L01~L10 | tutorials/workflow-101/lesson-01~10     | ✅         | 유지·번역 | P2   | 교육용 핵심 자료, 10편 일괄                                        |
| Simple Chatbot       | tutorials/simple-chatbot                | ✅         | 유지·번역 | P2   |                                                          |
| Twitter Chatflow     | tutorials/twitter-chatflow              | ❌         | **삭제** | -    | 이사님 결정(2026-06-04): 외부 연결 제거 (Twitter API 의존) |
| Customer Service Bot | tutorials/customer-service-bot          | ✅         | 유지·번역 | P2   |                                                          |
| AI Image Generation  | tutorials/build-ai-image-generation-app | ✅         | 유지·번역 | P2   | 튜토리얼 자체는 모델 종류와 무관(연결 방법 가이드). 사용자가 접근 가능한 이미지 생성 모델로 적용 |
| Article Reader       | tutorials/article-reader                | ✅         | 유지·번역 | P2   |                                                          |

## 액션별 집계 (2026-06-04 갱신)

| 액션 | 페이지 수 | 비율 |
|------|----------|------|
| **유지·번역** (그대로 번역) | ~69 | 68% |
| **부분 수정** (번역 + spx 차이 반영) | ~10 | 10% |
| **검토** | 0 | 0% |
| **삭제** (spx-agent 미보유 + 이사님 결정으로 제거) | 23 | 22% |
| **변환** (Dify 원본 → 신규 챕터로 대체) | 1 | 1% |
| **합계** | **~102** | 100% |

> 2026-06-04 이사님 결정 반영 (두 차례): 검토 27 → **0건** (1차에서 outbound 정리, 2차에서 inbound 3건도 유지·번역 확정), 삭제 5 → 23건, 변환 신설 1건. **모든 페이지 액션 확정 — Phase 4 본격 포팅 진입 가능**.

### 확정 삭제 (23건, 2026-06-04 갱신)

#### Phase 1.3 시점 (5/29 + 6/2)
| 페이지 | 삭제 사유 |
|--------|----------|
| publish/publish-to-marketplace | Marketplace 없음 |
| publish/webapp/web-app-access | Dify Enterprise 전용 기능 (`tag: "ENTERPRISE"`), spx-agent 미보유. Studio 내 앱 권한은 신규 챕터 "권한 설정"으로 대체 |
| knowledge/knowledge-request-rate-limit | Dify Cloud 전용 quota. spx-agent 미보유 |
| workspace/plugins | Marketplace 없음, 플러그인 관리 UI 없음 |
| workspace/subscription-management | SaaS 요금제 없음 |
| workspace/api-extension/cloudflare-worker | SaaS/Cloudflare 전용 |

#### 2026-06-04 이사님 결정 추가 (Marketplace/플러그인 제거 + 외부 연결 제거)
| 페이지 | 삭제 사유 |
|--------|----------|
| **nodes/trigger/plugin-trigger** | Marketplace + 플러그인 전체 제거 → Plugin Trigger 자동 비활성 |
| **monitor/integrations/integrate-langsmith** | 외부 SaaS observability 연동 제거 |
| **monitor/integrations/integrate-langfuse** | 〃 |
| **monitor/integrations/integrate-opik** | 〃 |
| **monitor/integrations/integrate-weave** | 〃 |
| **monitor/integrations/integrate-arize** | 〃 |
| **monitor/integrations/integrate-phoenix** | 〃 |
| **monitor/integrations/integrate-aliyun** | 〃 |
| **knowledge/.../sync-from-notion** | 외부 데이터 import 폐지 |
| **knowledge/.../sync-from-website** | 〃 |
| **knowledge/knowledge-pipeline/authorize-data-source** | 외부 데이터소스 인증 폐지 |
| **knowledge/connect-external-knowledge-base** | 외부 KB 연결 폐지 |
| **knowledge/external-knowledge-api** | 외부 KB API 스펙 폐지 |
| **tutorials/twitter-chatflow** | Twitter API 의존 |
| **workspace/api-extension/api-extension** | outbound 외부 endpoint 호출 (API Extension) — "외부 연결 제거" 적용 |
| **workspace/api-extension/external-data-tool-api-extension** | 〃 (워크플로에서 외부 endpoint로 데이터 fetch) |
| **workspace/api-extension/moderation-api-extension** | 〃 (콘텐츠 검열 외부 endpoint 송신) |

### 변환 (1건, 2026-06-04 추가)

| 페이지 | 변환 후 | 비고 |
|--------|--------|------|
| workspace/team-members-management | 신규 챕터 **사용자·부서 관리**로 대체 | KC는 "외부 시스템(예, 키클록)"으로 추상화. 본문은 신규 챕터 측에서 작성, 원본 페이지는 게시 안 함 |

### 검토 대기 (해소 완료, 2026-06-04)

2026-06-04 이사님 추가 결정으로 inbound 영역 3건도 유지·번역 확정. **검토 대기 0건**.

| 페이지 | 처리 결과 |
|--------|---------|
| publish/webapp/embedding-in-websites | ✅ 유지·번역 (inbound 유지 — 사내 다른 시스템에서 WebApp 임베드 가정) |
| publish/developing-with-apis | ✅ 유지·번역 (inbound 유지 — 사내 API 호출 가정) |
| knowledge/manage-knowledge/maintain-dataset-via-api | ✅ 유지·번역 (inbound 유지 — 사내 KB CRUD 가정) |

> **참고**: workspace/api-extension/* (3건)은 이전 표기에서 "양방향"으로 잘못 분류. 실제는 **outbound 일방향** (spx-agent가 사전 등록된 외부 endpoint에 요청 송신) → "외부 연결 제거" 적용 → 확정 삭제 표로 이동 완료 (2026-06-04).

#### 정리된 항목 (5/29~6/2 묶음 → 2026-06-04 확정)

<details>
<summary>이전 검토 묶음 (참고용)</summary>

| 질문 | 해당 페이지 | spx-agent 분석 결과 |
|------|-----------|-------------------|
| **MCP 지원 여부** | build/mcp, publish/publish-mcp | spx-agent 1.13.3에 MCP 기능 포함? | spx-agent RBAC에서 tool 타입에 `MCP Provider` 포함 → **MCP 코드 존재 확인됨**, 다만 실사용 여부는 이사님 확인 필요 |
| **플러그인 트리거 지원** | nodes/trigger/plugin-trigger | 플러그인 시스템 활성화 여부 | Marketplace 비활성화 가능성 높으나, 플러그인 코드 자체는 Dify 1.13.3에 포함. 실사용 여부 확인 필요 |
| **외부 서비스 trace 송출 (Monitor integrations)** | monitor/integrations/* (7) | 사내망에서 외부 SaaS observability(LangSmith/Langfuse/Opik/Weave/Arize/Phoenix/Aliyun)로 trace 송출 가능 여부 | 네트워크 정책 미확인. Langfuse/Phoenix는 self-hosted 옵션 있음 → 사내 배포 시 일부 살릴 수 있음 |
| **사내 망 외부 서비스 연동 (기타)** | tutorials/twitter-chatflow, embedding-in-websites | 외부 API 호출 가능 여부 (폐쇄망?) | 네트워크 정책 미확인 |
| **외부 데이터 import (지식, 패턴 a)** | knowledge/.../sync-from-notion, sync-from-website, authorize-data-source | 외부 데이터소스 → 사내 KB로 가져오기 가능 여부 | 네트워크 정책 미확인 |
| **외부 KB read-only 연결 (지식, 패턴 b)** | knowledge/connect-external-knowledge-base, knowledge/external-knowledge-api | 외부 KB(벡터DB 등)를 spx-agent에서 검색용으로 연결 가능 여부 | 사내 KB 인프라 보유 여부에 따라 결정 |
| **API Key 발급·외부 노출 (패턴 c 포함)** | publish/developing-with-apis, knowledge/maintain-dataset-via-api, workspace/api-extension/* (3) | API 키 발급 정책, 외부 시스템에서 spx-agent 자원 호출 허용 여부 | Dify 기본 API Key 기능은 코드에 그대로 존재. 사내 노출 정책만 확인 필요 |

→ **2026-06-04 이사님 결정으로 일괄 정리 완료** (위의 신규 묶음 5건만 남음)

</details>

### spx-agent 분석으로 확인된 부분 수정 보강 사항

기존 매핑표의 "부분 수정" 항목에 추가로 반영할 spx-agent 차이:

| 페이지 | 원래 액션 | 보강 내용 |
|--------|----------|----------|
| getting-started/introduction | 부분 수정 | Dify → SPX-Agent 소개, 신규 챕터 키워드 티저 (대시보드·감사로그·앱 권한 설정·사용자/부서 관리) |
| getting-started/quick-start | 부분 수정 | **로그인 = 사용자명/비밀번호 폼** (현재 개발된 화면 기준, KC SSO 리다이렉트 아님), 앱 생성 단계에서 부서 배정·권한 설정 한 줄 언급 |
| getting-started/key-concepts | 부분 수정 | **부서(Department), RBAC 개념만 추가**. 소유권/가시성/ACL은 신규 "권한 설정" 챕터에서 다룸 |
| workspace/readme | 부분 수정 | 부서 기반 workspace 구조 설명, 관리자 대시보드 언급 |
| workspace/personal-account-management | 부분 수정 | Keycloak이 계정 원천, 프로필 수정 범위 한정 |
| publish/README | 부분 수정 | Marketplace 게시 옵션 처리 = 전역 규칙 #5(보류) 따름. 권한에 따른 게시 제약은 신규 챕터 "권한 설정" 링크 |
| monitor/analysis | 부분 수정 | 제목 "Analysis" → **"모니터링"** (spx-agent UI 앱 설정 탭 표기). 신규 챕터 "대시보드"(관리자용 워크스페이스 KPI)와는 별개 영역(앱 단위 모니터링)임 명시. 각 통계 지표는 의미→읽는 법→업무 활용 3단 서술 ([[conventions]] §데이터 조회·시각화 챕터 서술 원칙) |

## 신규 추가 챕터 (spx-agent 전용) — 2026-05-29 v2

> spx-agent 코드베이스 분석 후 재구성. KC SSO 독립 챕터 삭제, RBAC → 권한 설정 + 사용자/부서 관리로 분리.

| 신규 챕터     | 경로(예정)                             | 출처                                                                                                     | 우선순위 | 페이지(추정) | 비고                                                              |
| --------- | ---------------------------------- | ------------------------------------------------------------------------------------------------------ | ---- | ------- | --------------------------------------------------------------- |
| 대시보드      | ko/use-spx-agent/analytics-audit/dashboard/ | `admin/` 컴포넌트 (kpi-section, dept-objects-chart, model-tokens-chart, dept-activity-table, drill-charts) | P1   | 2~3     | KPI 카드, 부서별 리소스, 모델 토큰, 드릴다운. Option α "통계·감사" top-level 그룹 안. ⚠️ **구조 나열 X** — 각 카드·차트·드릴다운을 의미→읽는 법→업무 활용 3단 서술 ([[conventions]] §데이터 조회·시각화) |
| 감사로그      | ko/use-spx-agent/analytics-audit/audit-log/ | `dify-audit/` 서비스 (13 collectors, EventTable, FilterBar, ExportButton)                                 | P1   | 2~3     | 이벤트 조회, 필터, 내보내기. Option α "통계·감사" top-level 그룹 안. ⚠️ **항목 나열 X** — 각 로그 항목·필터·내보내기를 의미→활용(감사·보안 시나리오) 3단 서술 ([[conventions]] §데이터 조회·시각화) |
| 권한 설정 (구 "앱 권한 설정") | ko/use-spx-agent/workspace/permissions/ | `app-permissions/`, `dataset-permissions/`, `tool-permissions/` 컴포넌트                                   | P1   | 1~2 (필요 시 확장)     | **2026-06-04 rename**: 앱·지식·도구 공통 권한 모델이라 "앱" 한정 표기 제거. 가시성 4단계, ACL 부여/취소, 소유권 이전. Option α "워크스페이스" 그룹 안 |
| 사용자/부서 관리 | ko/use-spx-agent/workspace/departments/ | `services/rbac/department_service.py`, Keycloak 그룹 동기화 + 스튜디오 헤더 좌측 부서 셀렉터(`web/app/components/header/account-dropdown/department-selector/`) | P1   | 2~3 (헤더 셀렉터 sub-section 포함) | **Dify 원본 workspace/team-members-management 변환·확장 (2026-06-04)**. KC는 본문에서 "외부 시스템(예, 키클록)"으로 추상화 ("관리 시스템에서 수행한다" 형식). 부서 CRUD, 멤버 배정, 권한 모델. Option α "워크스페이스" 그룹 안. **2026-06-05 추가: "헤더 부서 선택" sub-section 신설** (스튜디오 좌측 상단 드롭다운, 앱·지식 목록 부서별 필터링 — **도구는 미적용**, Tools 챕터에 한 줄 안내 cross-ref 필요. 권한 평가는 별개 — 부서 필터는 권한 통과 자원 중 `owner_department_id` 일치만 추가 좁힘). 분석 완료 → [[references/spx-department-filter-analysis]] |

**변경 이력**:
- ~~Keycloak SSO 로그인~~ → 삭제 (독립 챕터 불필요, 로그인 흐름은 Get Started/Quick Start에서 간단히 언급)
- ~~RBAC (통합)~~ → **권한 설정** + **사용자/부서 관리**로 분리 (사용자 관점: "내 자원 권한 어떻게 설정?" vs "부서 어떻게 관리?")
- ~~앱 권한 설정~~ → **권한 설정** rename (2026-06-04). A1 분석 결과 챕터 범위가 앱·지식·도구 공통 권한 모델 코어로 확정 → "앱" 한정 표기는 misleading
- ~~설정 사이드바 (KAN-29)~~ → **삭제** (2026-06-02). Dify 원본에도 존재하는 설정 다이얼로그에 토글 기능만 추가된 수준 → 별도 챕터 불필요. 필요 시 기존 페이지(`web-app-settings`, `personal-account-management` 등) 부분 수정에서 한 줄 언급
- **사용자/부서 관리 P2 → P1 격상** (2026-06-04): Dify 원본 team-members-management 자리를 대체하는 위치라 시급도 ↑. KC 추상화 표기 정책 박제

**신규 합계: 8~12 페이지** (설정 사이드바 1~2페이지 차감)

### 신규 챕터 ↔ spx-agent 코드 매핑

| 신규 챕터 | 백엔드 소스 | 프론트엔드 소스 |
|----------|-----------|---------------|
| 대시보드 | `api/controllers/console/dashboard/` (DashboardKpiService 등) | `web/app/components/admin/` (kpi-section, drill-charts 등) |
| 감사로그 | `dify-audit/src/lib/collectors/` (13 collectors) | `dify-audit/src/components/audit/` (EventTable, FilterBar) |
| 권한 설정 | `api/services/rbac/authorization_service.py`, `resource_permission_service.py` | `web/app/components/app-permissions/`, `dataset-permissions/`, `tool-permissions/` |
| 사용자/부서 관리 | `api/services/rbac/department_service.py`, `api/libs/keycloak_admin.py` | `web/app/components/account-setting/` (멤버·부서 탭), `web/app/components/header/account-dropdown/department-selector/` (헤더 셀렉터 — 2026-06-05 신규 박제, `useActorDepartments` + `useSelectedDepartmentStore` zustand persist) |

## 최종 산출 예상 (v2)

| 구분 | 페이지 수 |
|------|----------|
| 원본 유지·번역 + 부분 수정 | ~70 (삭제 5건 + 검토 결과 일부 추가 삭제 가능) |
| 검토 후 유지 (낙관) | ~20 |
| 검토 후 삭제 (비관) | ~20 |
| 신규 추가 | ~10 |
| **최종 예상 범위** | **80~100 페이지** |

## 후속 트리거

- ~~Phase 1.1 완료 시 본 표 챕터 행 박기~~ → 완료 (2026-05-29)
- Phase 3 컨펌 후 검토 항목 확정 + 우선순위 재조정
- Phase 4 본격 포팅 시 본 표에 진행 상태 표시
