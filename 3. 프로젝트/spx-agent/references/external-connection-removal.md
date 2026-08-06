---
tags: [프로젝트, dify, AI-Agent, 폐쇄망, 외부연결]
type: reference
date: 2026-06-08
last_updated: 2026-06-08
purpose: spx-agent-docs에서 확정한 "외부 연결 제거(폐쇄망 가정)" 정책을 실제 spx-agent 코드/UI에 반영하기 위한 코드 위치 매핑 + 처리 정책 박제. 진행 상태·체크리스트는 SESSION_HISTORY가 담당.
---

# 외부 연결 제거 — spx-agent 코드 매핑 (references)

> **"왜 빼는지"는 spx-agent-docs가 단일 진실(SoT)**: [[3. 프로젝트/spx-agent-docs/docs/decisions.md]] 2026-06-04 §결정 1~5 + [[3. 프로젝트/spx-agent-docs/docs/references/external-connection-framing-proposal.md]].
> **"지금 어디까지 했나"(진행 상태)는 [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]]** 가 담당.
> 본 문서는 그 사이 — **"외부연결이 제품 코드 어디에 있고, 어떻게 처리하기로 했나"** 라는 영속 매핑만 박는다. 작업이 끝나도 "그 기능이 어디 있었는지" 들춰볼 참고 자산.

## § 1. 배경

- spx-agent-docs(사용자 매뉴얼)에서는 2026-06-04 이사님 회의로 외부 연결 관련 페이지를 ❌ 삭제 처리 완료.
- 그러나 **실제 spx-agent 제품에는 해당 기능이 여전히 노출**돼 있음 (Dify 원본 그대로) → docs와 제품 불일치.
- 본 작업: 폐쇄망(사내망) 환경 가정에 맞춰 **제품 코드/UI에서도 외부 연결을 제거/비활성화**해 docs와 정합.

## § 2. 제거/유지 정책 (docs 결정 그대로 — 재논의 금지)

### ❌ 제거 대상

| #   | 항목                                                                             | docs 근거 | 비고                                     |
| --- | ------------------------------------------------------------------------------ | ------- | -------------------------------------- |
| 1   | Marketplace UI + 플러그인 install 시스템                                              | 결정 1    | 사내 소스로 미리 심는 모델, 사용자 install 아님        |
| 2   | Plugin Trigger 노드 (langbot/lark/telegram/outlook/gmail 등)                      | 결정 1    | Marketplace 의존 트리거. 빌트인(일정/웹훅 트리거)은 유지 |
| 3   | Monitor 송출 10종 전체 (LangSmith/Langfuse/Opik/Weave/Arize/Phoenix/Aliyun + mlflow/databricks/tencent — 6/8 결정)      | 결정 2    | outbound trace 송출                      |
| 4   | 지식 외부 import 3종 (Sync from Notion / Sync from Website / Authorize Data Source) | 결정 2    | 외부 → 내부 데이터 복사                         |
| 5   | 외부 KB 연결 2종 (Connect External KB / External Knowledge API)                     | 결정 2    | 외부 KB read-only 연결                     |
| 6   | Twitter 연동 (Twitter Chatflow 류)                                                | 결정 2    | 외부 API 호출                              |
| 7   | API Extension (workspace/api-extension)                                        | 결정 2    | outbound 호출                            |

### ✅ 유지 (혼동 주의 — 빼지 말 것)

| 항목 | docs 근거 | 이유 |
|------|----------|------|
| 모델 제공자 설치 (워크스페이스 > 모델) | 결정 1 | 환경마다 백엔드 다름 (vLLM/Ollama/상용) |
| MCP (build/mcp, publish-mcp) | 결정 3 | 사내·사외 MCP 서버 모두 연동 가능 |
| inbound 3종 (웹사이트 임베드 / API로 호출 / maintain-dataset-via-api) | 결정 4 | 사내 타 시스템이 spx-agent를 호출하는 시나리오 |
| Keycloak 등 외부 IdP | 결정 5 | 제거 아님 — "외부 시스템" 추상화는 docs 한정 (제품은 유지) |

## § 3. 코드 위치 매핑 (조사 완료 — 2026-06-08, 4배치)

> 제거 대상 7종을 4배치로 전수 조사 완료. 처리 방식·미결정은 § 4 참조.

| #   | 항목                       | 프론트(web) 위치                                                                                                                                                                      | 백엔드(api) 위치                                                                                                                                                                                     | 처리 방식(숨김/플래그/삭제)                                                                                                                                                  |
| --- | ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Marketplace UI + install | `(commonLayout)/plugins/page.tsx`, `header/plugins-nav/`, `components/plugins/marketplace/`                                                                                      | `controllers/console/workspace/plugin.py`, `core/helper/marketplace.py`, `services/feature_service.py`                                                                                          | ✅ env `MARKETPLACE_ENABLED=false` (코어 수정 0). 단 `/plugins` 메뉴·페이지 진입은 별도 숨김 필요                                                                                     |
| 2   | Plugin Trigger 노드        | `components/workflow/block-selector/featured-triggers.tsx`                                                                                                                       | `controllers/console/workspace/trigger_providers.py`, `core/plugin/impl/trigger.py`                                                                                                             | 플러그인 미설치 시 자연 미노출. 빌트인(schedule/webhook) 유지. 기존 설치분은 별도 정리                                                                                                        |
| 3   | Monitor 송출 10종 전체        | `app/[appId]/overview/tracing/` (`type.ts` enum, `config-popup.tsx`, `panel.tsx`)                                                                                                | `core/ops/entities/config_entity.py`(`TracingProviderEnum`), `core/ops/ops_trace_manager.py`(`OpsTraceProviderConfigMap`), `core/ops/{provider}_trace/`, `controllers/console/app/ops_trace.py` | ❌ **환경변수 없음 → 코어 수정 필요** (enum + provider_config_map + UI 3곳에서 10종 전부 제거). 목록이 하드코딩 분산                                                                            |
| 4   | 지식 외부 import 3종          | `datasets/create/step-one/components/data-source-type-selector.tsx`, `datasets/create/website/*`, `base/notion-page-selector/*`, `header/account-setting/data-source-page-new/*` | `controllers/console/datasets/data_source.py`, `.../website.py`, `controllers/console/auth/data_source_oauth.py`, `libs/oauth_data_source.py`                                                   | 혼합: Notion=env `NOTION_*` 비움 / Website=프론트 `NEXT_PUBLIC_ENABLE_WEBSITE_FIRECRAWL/JINAREADER/WATERCRAWL=false`. ⚠️ Website 백엔드 라우트(`website.py`)는 플래그 없음 → API는 잔존 |
| 5   | 외부 KB 연결 2종              | `datasets/external-knowledge-base/*`, `datasets/external-api/*`, `context/external-knowledge-api-context.tsx`                                                                    | `controllers/console/datasets/external.py`, `services/external_knowledge_service.py`                                                                                                            | ❌ **환경변수 없음 → 코어 수정 필요** (라우트·UI 제거 또는 게이트)                                                                                                                       |
| 6   | Twitter 연동               | 없음                                                                                                                                                                               | 없음                                                                                                                                                                                              | ✅ **제품 코드에 Twitter 전용 코드 없음** (docs 튜토리얼/HTTP 노드 수준) → 코드 작업 불필요                                                                                                  |
| 7   | API Extension            | `header/account-setting/index.tsx`(메뉴), `.../api-based-extension-page/*`                                                                                                         | `controllers/console/extension.py`, `services/api_based_extension_service.py`, `core/extension/api_based_extension_requestor.py`, `models/api_based_extension.py`                               | ❌ env 없음 → 코어 수정. ⚠️ Moderation·External Data Tool의 "api 모드"가 의존 → 메뉴 숨김 + 라우트 차단 + factory에서 api 타입만 거부(다른 모드 유지)                                                |

### 배치 1 상세 (Marketplace + Plugin Trigger) — 2026-06-08 조사

- **Marketplace 끄기 (핵심)**: `api/configs/feature/__init__.py` `MarketplaceConfig` → env `MARKETPLACE_ENABLED`(기본 true) / `MARKETPLACE_API_URL`(기본 `marketplace.dify.ai`). `feature_service.py`가 `/console/api/system-features`로 `enable_marketplace` 내려보냄 → 프론트 `useGlobalPublicStore(s => s.systemFeatures)`가 설치 옵션 숨김. **`MARKETPLACE_ENABLED=false` 하나로 코어 수정 0.**
- **단, 남는 노출**: 위 플래그를 꺼도 `/plugins` 페이지와 헤더 메뉴(`header/plugins-nav/`)는 **여전히 접근 가능**(install 옵션만 숨김). 마켓 진입 자체를 없애려면 프론트 nav 제거가 추가로 필요 → **미결정 ①**.
- **추가 통제 플래그**: `plugin_installation_permission.restrict_to_marketplace_only` (`feature_service.py`). github 업로드 설치 경로(`plugin/upload/github`)도 존재 → 설치 출처 제한 시 같이 검토.
- **Plugin Trigger**: Marketplace와 **독립**. plugin daemon에서 동적 로드(`core/plugin/impl/trigger.py` → `plugin/{tenant_id}/management/triggers`). 마켓 끄면 신규 설치만 막힘 → **이미 설치된 플러그인 트리거는 계속 노출** → 완전 제거 시 설치분 정리 별도 → **미결정 ②**.
- **유지 항목 안전 ✅**: MCP(`core/mcp/`, `controllers/console/app/mcp_server.py`)·모델 제공자는 플러그인 시스템과 코드 분리 → `MARKETPLACE_ENABLED=false` 영향 없음.

### 배치 2 상세 (지식 외부 import + 외부 KB 연결) — 2026-06-08 조사

- **데이터 소스 선택 화면**: `data-source-type-selector.tsx`에 내부(파일 업로드/텍스트)와 외부(Notion/Website)가 한 목록. 외부 옵션만 빠지고 내부는 남는 구조 → 내부 KB 본체(파일 업로드·텍스트·임베딩·검색)는 **독립, 유지 안전 ✅**.
- **Notion (import + Authorize Data Source)**: env `NOTION_INTEGRATION_TYPE` / `NOTION_CLIENT_ID` / `NOTION_CLIENT_SECRET` / `NOTION_INTERNAL_SECRET` (`api/configs/extra/notion_config.py`). 비우면 OAuth provider 목록(`data_source_oauth.py`)에서 notion 빠짐. **코어 수정 0** 가능.
- **Website 크롤링 (Firecrawl/Jina/Watercrawl)**: 프론트는 `NEXT_PUBLIC_ENABLE_WEBSITE_*=false`(`web/env.ts`, `web/config/index.ts`)로 선택지 숨김. **그러나 백엔드 `controllers/console/datasets/website.py` 라우트엔 플래그가 없어 API는 살아있음** → 완전 차단하려면 백엔드 게이트/라우트 처리 필요.
- **외부 KB 연결 2종 (Connect External KB / External Knowledge API)**: ⚠️ **환경변수 자체가 없음**. `external.py` 라우트(`/datasets/external-knowledge-api`)와 `external-knowledge-base/`·`external-api/` 프론트를 직접 제거/게이트해야 함. DB 테이블 `ExternalKnowledgeApis`/`ExternalKnowledgeBindings` 존재 → 스키마는 건드릴 필요 없고 진입만 차단하면 됨.
- **➡️ 첫 코어 수정 지점**: 배치 2에서 "플래그로 못 끄는" 항목(Website 백엔드, 외부 KB 2종)이 나옴. 처리 방식(피처플래그 신설 vs 라우트 제거 vs UI 숨김만)을 구현 단계에서 결정해야 함 → **미결정 ③**.

### 배치 3 상세 (Monitor integrations) — 2026-06-08 조사

- **끄는 법**: ❌ 환경변수·플래그 없음. 제공자 목록이 **enum + config map + UI에 하드코딩 분산** → 7종 제거하려면 ① `config_entity.py` `TracingProviderEnum`, ② `ops_trace_manager.py` `OpsTraceProviderConfigMap.__getitem__` case절, ③ 프론트 `type.ts`/`config-popup.tsx`/`panel.tsx` 세 군데를 모두 수정. (앱별 `App.tracing.enabled=false`는 런타임 상태일 뿐 제공자 노출은 안 막음.)
- **⚠️ docs에 없던 추가 송출 제공자 3종 발견 → 전부 제거 확정**: 코드에는 7종 외 **mlflow / databricks / tencent**도 있음(총 10종). 셋 다 외부 SaaS 송출 → **2026-06-08 사용자 결정: 10종 전부 제거**. ⏳ 후속: spx-agent-docs `decisions.md` 결정 2에 "10종" 피드백 박을 것.
- **유지 안전 ✅**: 내부 모니터링/로그/Analysis 화면, 감사로그(dify-audit), 대시보드는 `core/ops/`와 무관 → trace 송출만 빠지고 내부 조회 화면은 남음.

### 배치 4 상세 (API Extension + Twitter) — 2026-06-08 조사

- **Twitter**: 백엔드·프론트 전수 grep 결과 "twitter" 0건. **제품 코드에 전용 코드 없음** → docs 튜토리얼(HTTP Request 노드로 외부 API 호출 가이드) 수준. 코드 작업 불필요.
- **API Extension**: env 없음 → 코어 수정. 메뉴(`account-setting/index.tsx`)+UI(`api-based-extension-page/`)+라우트(`controllers/console/extension.py`)+외부호출(`core/extension/api_based_extension_requestor.py`).
- **⚠️ 간섭 주의**: `APIBasedExtensionPoint` enum이 **Moderation(콘텐츠 검열)** 과 **External Data Tool** 의 "api 모드" 백엔드로 쓰임 (`core/moderation/api/api.py`, `core/external_data_tool/api/api.py`). 단, 이 두 기능엔 다른 모드(keywords/openai 등)도 있음 → **"api 모드"만 거부**하면 기능 자체는 유지 가능.
- **권장 처리**: ① 메뉴 숨김 ② 라우트 차단 ③ `core/moderation/factory.py`·`core/external_data_tool/factory.py`에서 "api" 타입 거부. DB 테이블·requestor는 남겨도 런타임 격리됨.

## § 4. 조사 종합 — 처리 난이도 분류 + 미결정

### 처리 난이도 3분류

| 난이도 | 항목 | 방식 |
|--------|------|------|
| 🟢 **플래그로 끔 (코어 수정 0)** | Marketplace, Notion import/인증, Website 크롤링(프론트) | env `MARKETPLACE_ENABLED=false` / `NOTION_*` 비움 / `NEXT_PUBLIC_ENABLE_WEBSITE_*=false` |
| 🔴 **코어 수정 필요** | Monitor 트레이싱 10종 전체, 외부 KB 연결 2종, Website 백엔드 라우트, API Extension | enum/목록/라우트/factory 직접 수정 (env 없음) |
| ⚪ **작업 불필요** | Twitter | 제품 코드에 없음 |

### 🔴 코어 수정 규모 산정 (2026-06-08)

> 방식에 따라 규모가 크게 갈림. 같은 작업도 "직접 삭제"면 큰 작업, "피처플래그 분기"면 작은 작업.

| 방식 | 규모 | 위험 | Dify 무수정 원칙 |
|------|------|------|----------------|
| **직접 제거** (코드·컴포넌트 삭제) | 24개 파일(원본 Dify 11개), 통째 삭제 시 ~2천 줄 규모, 외부 KB는 **DB 마이그레이션 동반** | 중상 (7/10) | ❌ 위반, 업스트림 대조·재적용 비용 큼 |
| **피처플래그** (env + 분기) | env 4개(`ENABLE_TRACING`·`ENABLE_EXTERNAL_KB`·`ENABLE_WEBSITE_CRAWL`·`ENABLE_API_EXTENSION`) + 항목당 조건문 3~5줄 = **원본 ~20줄 수정** (트레이싱 타입 정의만 일부 제거 불가피) | 하 (3/10) | ✅ 부합, 가역적, 업그레이드 충돌 최소 |

- **항목별 플래그 차단 가능성**: Website 90% / API Extension 85% / 외부 KB 70% / 트레이싱 30%(타입 정의는 못 빼지만 **라우트 차단 + overview에서 tracing 패널 미렌더로 진입 실질 차단**).
- **결론**: 외부 KB·Website·API Extension은 플래그로 거의 다 가려지고, 트레이싱만 타입 일부를 손대야 함. 전체적으로 **피처플래그 방식이면 원본 수정 최소 + 가역적**.

### 미결정 (구현 단계에서 결정)

- **① `/plugins` 메뉴 진입**: 플래그 꺼도 페이지·헤더 메뉴는 남음 → 메뉴 자체 숨길지.
- **② 기존 설치 플러그인**: 이미 설치된 플러그인 트리거는 계속 노출 → 정리할지.
- **③ 코어 수정 방식 통일**: 🔴 항목들을 (a) 피처플래그 신설 / (b) 라우트·목록 직접 제거 / (c) UI 숨김만 중 어느 패턴으로 갈지. → 규모 산정 결과 **(a) 피처플래그 권장**(원본 ~20줄 vs 직접 삭제 ~2천 줄, Dify 무수정 원칙 부합). 사용자 확인 대기.
- **✅ ④ 결정됨 (2026-06-08): Monitor 송출 10종 전부 제거** — docs 7종 + 코드 추가 3종(mlflow/databricks/tencent) 모두 외부 SaaS 송출이라 전부 제거. ⏳ 후속: spx-agent-docs `decisions.md` 결정 2에 "10종" 피드백.

## § 5. 관련 문서

- 정책 SoT: [[3. 프로젝트/spx-agent-docs/docs/decisions.md]] (2026-06-04 §결정 1~5)
- framing 상세: [[3. 프로젝트/spx-agent-docs/docs/references/external-connection-framing-proposal.md]]
- 진행 상태: [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (2026-06-08 §)
- spx-agent 진입점: [[3. 프로젝트/spx-agent/CLAUDE.md]]
