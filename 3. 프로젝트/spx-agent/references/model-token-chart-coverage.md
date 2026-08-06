---
tags: [프로젝트, dify, AI-Agent, references]
type: references
date: 2026-06-11
external_dependency_observed_at: 2026-06-11
purpose: 모델별 토큰 차트의 "모델 타입 커버리지" 확장 조사 — 임베딩/리랭커/STT/TTS/moderation을 차트에 넣을 수 있나 + 권고안
related: [workflow-model-classification.md, audit-details-spec.md, dify-app-modes.md, dify-db-schema.md, data-mart.md, model-tokens]
verified_from: graphon model_runtime 패키지 + api/core 직접 읽기 + dify-audit collectors + 운영 DB(192.168.10.159) 읽기전용 SELECT
defects: [H-DASH-02, H-DASH-09]
---

# 모델별 토큰 차트 — 모델 타입 커버리지 조사

> **한 줄 결론**
> 현재 차트 = "채팅 답변 LLM(text-generation) 비용"뿐. **6개 model_type 중 토큰 차트에 의미 있는 타입은 LLM·임베딩 2종뿐**이고, 나머지 4종(rerank/STT/TTS/moderation)은 **graphon invoke 반환값에 usage(토큰) 필드 자체가 없어 구조적 불가**.
> → **현실적 확장 = 임베딩 인덱싱 토큰 1종.** collector에 dataset JOIN 한 줄 + 마트 UNION 한 갈래로 편입 가능(저비용·고가치). 운영 DB에 임베딩 토큰 44.5만 + 유료 gemini-embedding 실재 → 편입 실익 확인.

> ⚠️ 본 조사는 **커버리지(차트가 어떤 타입을 보여줄 것인가)** 결정용. H-DASH-09(로컬/원격 판별 — 어디서 도나)와 **다른 트랙**. 섞지 말 것.

---

## 0. 현재 베이스라인 (B안 반영 후)

조사 시점(`KAN-29-admin-dashboard` 브랜치) 이미 **B안(워크플로/챗플로 LLM 노드 재분류)**이 들어가 있음:

- collector `workflow-nodes.ts:66-109` — `process_data`에서 `llm`/`question-classifier`/`parameter-extractor` 3종 노드의 `model_provider`/`model_name`/`usage.total_tokens` 추출 (`MODEL_NODE_TYPES` set, line 8).
- 마트 `20260610000000_rewrite_model_tokens_daily_union/migration.sql` — `spx_mv_model_tokens_daily`를 UNION ALL로 재작성:
  - (1) `message_send` AND `app_mode_d <> 'advanced-chat'` (비챗플로우 메시지)
  - (2) `workflow_node_execute` AND `app_mode_d IN ('workflow','advanced-chat')` (노드)
- 서비스 `dashboard_model_tokens_service.py` — UNION 마트 SELECT, `LOCAL_PROVIDERS`(ollama/xinference/localai) 로컬 판별.

**즉 Phase 0(현재) 커버리지 = 답변 LLM + 워크플로/챗플로 LLM 노드.** 전부 `model_type = LLM(text-generation)` 계열. 본 조사는 그 위에 **비-LLM 타입 확장**을 얹는 그림.

---

## 1. 조사 1 — Dify model_type 전수 + 토큰 소비 여부

### 1.1 model_type enum (전수)

`api/.venv/Lib/site-packages/graphon/model_runtime/entities/model_entities.py:12-23` `ModelType(StrEnum)`:

```python
class ModelType(StrEnum):
    LLM = auto()                       # "llm"  (origin: "text-generation")
    TEXT_EMBEDDING = "text-embedding"  # (origin: "embeddings")
    RERANK = auto()                    # "rerank" (origin: "reranking")
    SPEECH2TEXT = auto()               # "speech2text"
    MODERATION = auto()                # "moderation"
    TTS = auto()                       # "tts"
```

→ **6종.** `value_of`/`to_origin_model_type`(24-63행)이 `text-generation↔llm`, `embeddings↔text-embedding`, `reranking↔rerank` 별칭 매핑.

### 1.2 타입별 토큰 소비/기록 여부 (graphon invoke 반환 엔티티 근거)

| model_type | invoke 반환 타입 | usage(토큰) 필드 | 과금 단위 | 근거 파일:라인 |
|---|---|:---:|---|---|
| **LLM** | `LLMResult` | ✅ `LLMUsage.total_tokens` | 토큰 | `entities/llm_entities.py:48` |
| **TEXT_EMBEDDING** | `EmbeddingResult` | ✅ `EmbeddingUsage.tokens/total_tokens` | 토큰 | `entities/text_embedding_entities.py:16-37` |
| **RERANK** | `RerankResult` | ❌ (model+docs만, usage 없음) | 문서/호출 | `entities/rerank_entities.py:21-28` |
| **SPEECH2TEXT** | `str` (텍스트) | ❌ | 오디오 초 | `model_providers/__base/speech2text_model.py:14-31` |
| **TTS** | `Iterable[bytes]` (오디오) | ❌ | 입력 문자 | `model_providers/__base/tts_model.py:17-42` |
| **MODERATION** | `bool` (flagged) | ❌ | 입력 문자 | `model_providers/__base/moderation_model.py:14-33` |

> **결정적 사실**: 토큰(`usage.total_tokens`) 개념을 가진 타입은 **LLM·임베딩 2종뿐**. rerank/STT/TTS/moderation은 invoke 반환 엔티티에 usage가 **정의조차 안 됨** → "토큰 차트"에 원천적으로 부적합(1차 후보 탈락).

---

## 2. 조사 2 — 타입별 (a)사용이벤트 (b)토큰 (c)모델식별자 가용성 매트릭스

> 범례: ✅가용 / 🔶top-level·간접(JOIN/별도컬럼) / ⚠️부분·조건부 / ❌없음
> 차트 편입 조건 = **(a)(b)(c) 3요소가 다 있어야** 함.

| 타입 / 채널 | (a) 사용 이벤트 | (b) 토큰 | (c) 모델 식별자 | 근거 |
|---|:---:|:---:|:---:|---|
| **답변 LLM** (chat/completion/agent-chat) `message_send` | ✅ `messages` 행 | ✅ `message_tokens`+`answer_tokens` | ✅ `model_provider`/`model_id` | `collectors/messages.ts`, 기준선 |
| **워크플로/챗플로 LLM 노드** `workflow_node_execute` | ✅ B안으로 유입 | ✅ `process_data.usage.total_tokens` | ✅ `process_data.model_provider`/`model_name` | `llm/node.py:309-319`, `workflow-nodes.ts:72-80` |
| **임베딩 — 인덱싱** `document_upload` | ✅ `documents` 행 | ✅ `documents.tokens` | 🔶 `datasets.embedding_model`(_provider) — **dataset 단위, JOIN 필요** | `models/dataset.py:148-149,470`, `core/indexing_runner.py:581-587,694-697`, `collectors/documents.ts:32-36` |
| **임베딩 — 쿼리타임 RAG** | ✅ `dataset_queries` 행 | ❌ `EmbeddingUsage` 계산되나 **미저장** | ❌ | `models/dataset.py:1057-1080`, `core/rag/retrieval/dataset_retrieval.py:1042-1053`, `core/rag/embedding/cached_embedding.py:203-207` |
| **리랭커** | ⚠️ 검색 파이프라인 인메모리, **영속 이벤트 없음** | ❌ `RerankResult` usage 없음 | 🔶 `datasets.retrieval_model` JSON | `core/rag/datasource/retrieval_service.py:833-846`, `core/rag/rerank/rerank_model.py:104-106`, `models/dataset.py:152,276-284` |
| **knowledge-retrieval 노드** `workflow_node_execute` | ✅ (노드 행 존재) | ⚠️ `process_data.usage`는 **metadata-filter LLM usage**뿐(임베딩/rerank 토큰 아님) | ❌ `model_provider/name` 미기록 | `core/workflow/nodes/knowledge_retrieval/knowledge_retrieval_node.py:122-132,212-216` |
| **STT / TTS / moderation** | ❌ 영속 이벤트 없음 | ❌ (토큰 개념 부재) | ❌ | §1.2 + `core/moderation/input_moderation.py:48-60`(trace만) |

### 2.1 매트릭스 핵심 판독

- **임베딩 인덱싱만 "3요소 거의 충족"** — (a)(b) 완비, (c)는 dataset 단위로 존재하나 collector가 아직 안 가져옴(🔶). **유일하게 보강으로 메꿔지는 칸.**
- **임베딩 쿼리타임**은 (b)(c) 둘 다 빠짐. `cached_embedding.py:203-207`에서 `embedding_result.usage`가 반환되지만 코드가 **버림**(미저장). `dataset_queries`에 토큰/모델 컬럼 없음 → collector로 못 메꿈(Dify 코어 수정 영역).
- **리랭커**는 (b)가 graphon 레벨에서 부재(`RerankResult`에 usage 없음) → 코어를 고쳐도 토큰을 만들 수 없음(reranking 자체가 토큰 과금이 아님).
- **STT/TTS/moderation**은 (a)(b)(c) 전부 ❌ + 토큰 개념 부재.

---

## 3. 조사 3 — collector 보강 필요량

### 3.1 ✅ 보강으로 해결 (임베딩 인덱싱) — (c)만 빠짐

`documents.ts`는 **이미 `datasets ds` LEFT JOIN 보유**(line 39) + `documents.tokens` SELECT(line 32). (c) 모델식별자만 추가하면 됨. B안 `app.mode` 보강과 **동일 패턴**:

```diff
  SELECT
    d.id::text as id, d.name, d.dataset_id::text as dataset_id,
    ds.name as dataset_name, d.indexing_status, d.error, d.tokens,
-   d.data_source_type, ds.tenant_id::text as tenant_id
+   d.data_source_type, ds.tenant_id::text as tenant_id,
+   ds.embedding_model, ds.embedding_model_provider
  FROM documents d
  LEFT JOIN datasets ds ON d.dataset_id = ds.id
```
```diff
    details: {
      datasetId: r.dataset_id, datasetName: r.dataset_name,
      indexingStatus: r.indexing_status, error: r.error,
-     tokens: r.tokens, dataSourceType: r.data_source_type,
+     tokens: r.tokens, dataSourceType: r.data_source_type,
+     modelProvider: r.embedding_model_provider,
+     modelId: r.embedding_model,
+     totalTokens: r.tokens,   // generated col model_tokens 차트 진입용 (tokens 별칭)
    },
```

- **변경량 ~6줄**(타입 정의 +2 포함 ~8줄). JOIN 추가 0건.
- `details.modelProvider`/`modelId`/`totalTokens` 키가 채워지면 **기존 Generated Column(`model_provider_d`/`model_id_d`/`total_tokens_d`)이 자동 추출** → 새 ALTER 불필요(B안과 동일 메커니즘).
- ⚠️ `appMode` 보강은 불필요/무의미(document_upload는 앱 모드 무관). 마트 UNION 갈래(3)은 `action='document_upload'`로 직접 필터.
- ⚠️ **백필**: 기존 `document_upload` 행엔 모델 키 없음 → 재수집(커서 리셋) 또는 details UPDATE 필요. 운영 DB 문서 11건뿐이라 백필 비용 무시 가능.

### 3.2 ❌ 보강 불가 (구조적 한계) — (b) 또는 (b)(c) 동시 결손

| 타입 | 빠진 요소 | 왜 collector로 못 메꾸나 |
|---|---|---|
| **임베딩 쿼리타임** | (b)토큰 + (c)모델 | `dataset_queries` 테이블에 토큰/모델 컬럼이 **없음**. `cached_embedding.py`가 `usage`를 버림 → **Dify 코어 수정 + 스키마 컬럼 추가** 필요(무수정 원칙 위배). |
| **리랭커** | (b)토큰 | `RerankResult`(graphon)에 usage 필드 자체가 없음 → 코어를 고쳐도 토큰 생성 불가. |
| **STT/TTS/moderation** | (a)(b)(c) 전부 | invoke 반환에 usage 없음 + 영속 이벤트 없음. graphon 엔티티 재설계 수준. |

---

## 4. 조사 4 — 마트/차트 설계 변경 윤곽 (임베딩 인덱싱 편입 기준)

### 4.1 model_type 차원 추가

`spx_mv_model_tokens_daily`에 `model_type` 컬럼 추가 + UNION 각 갈래에 리터럴 부여:

```sql
-- (1)(2) LLM 갈래
SELECT tenant_id, occurred_at, model_provider_d, model_id_d, total_tokens_d,
       'llm'::text AS model_type
FROM public.spx_audit_events WHERE action='message_send' ...
UNION ALL ... action='workflow_node_execute' ...
UNION ALL
-- (3) 임베딩 인덱싱 갈래 (신규)
SELECT tenant_id, occurred_at, model_provider_d, model_id_d, total_tokens_d,
       'embedding'::text AS model_type
FROM public.spx_audit_events
WHERE action='document_upload' AND model_id_d IS NOT NULL AND total_tokens_d IS NOT NULL
```
- 바깥 `GROUP BY 1,2,3,4, model_type` + UNIQUE INDEX에 `model_type` 추가.
- **카디널리티 영향 미미**: (모델 × 일 × 타입). 운영 임베딩 모델 4종뿐.
- `api/models/mart.py` `MvModelTokensDaily`에 `model_type` 컬럼 추가(PK 확장).
- ⚠️ document_upload는 **디버그 필터 무관**(invoke_from/triggered_from 키 없음) → `NOT COALESCE(...)` 갈래는 자동 통과(NULL→FALSE). 그대로 두면 됨.

### 4.2 소스 UNION

현재 2갈래(message_send + workflow_node_execute)에 **(3) document_upload 추가**. 패턴 동일, drift 주의점은 §6.

### 4.3 차트 UX — 임베딩 vs LLM 분리 권장

| 축 | 답변/노드 LLM | 임베딩 인덱싱 |
|---|---|---|
| 비용 성격 | 대화당 **반복 누적** | 문서 업로드 시 **1회성 burst** |
| 일별 분포 | 고른 트래픽 | 특정 업로드일 **스파이크**(운영 max 12.6만/문서) |
| 단위 의미 | "사용량" | "색인 적재량" |

→ **같은 막대에 섞으면 오해**(임베딩 burst가 특정일 LLM 비용처럼 보임). **권장**: ① `model_type` 필터/탭으로 분리, 또는 ② 색·범례 구분(현재 `model-tokens-chart/index.tsx`는 단일 막대 + 로컬 teal만 — `model_type`별 색/그룹 추가 여지). PM 결정 필요(§7).

### 4.4 영향 받는 코드

- `dify-audit/src/lib/collectors/documents.ts` (+6줄, §3.1)
- `dify-audit/prisma/audit/migrations/2026XXXX_add_embedding_to_model_tokens/` (신규 마이그레이션 — UNION 재작성, idempotent §6)
- `api/models/mart.py` `MvModelTokensDaily` (+`model_type` 컬럼)
- `api/services/admin/dashboard_model_tokens_service.py` (+`model_type` 그룹/응답 필드)
- `api/services/admin/dashboard_drill_calls_service.py` `get_model_call_share` (모델 차원 2종 정합 — 동일 마트 공급)
- `web/.../model-tokens-chart/index.tsx` (타입 분리 UX)

> ⚠️ 모델 **호출 점유**(`get_model_call_share`)는 임베딩 인덱싱 `calls`도 같이 들어옴 → "호출"의 의미가 더 흐려짐(대화 호출 vs 문서 색인). model-call-share에는 임베딩을 넣지 않거나 별도 표기 권장.

---

## 5. 조사 5 — 운영 현황 검증 (192.168.10.159, 읽기전용 SELECT)

> 접속: `postgresql://postgres:***@192.168.10.159:5432/dify` (DBeaver "dify 3"). dify-audit 생성 Prisma 클라이언트로 datasource URL 오버라이드, **SELECT only**. 일회성 probe 스크립트는 실행 후 삭제(자격증명 하드코딩 회피).

### 5.1 등록된 model_type 분포 (`provider_models`)

```
text-generation : 9
embeddings      : 4
```
→ **rerank / speech2text / tts / moderation 등록 0건.** 사내는 실제로 **LLM + 임베딩 2종만** 등록·운영. 구조적 불가 4종은 **실익도 0**(이중으로 확인).

### 5.2 실사용 임베딩 모델 (`datasets.embedding_model`)

```
langgenius/ollama/ollama              | bge-m3              | 5 datasets
langgenius/openai_api_compatible/...  | bge-m3-ko          | 4
langgenius/gemini/google              | gemini-embedding-001 | 1   ← 유료 외부 API
langgenius/ollama/ollama              | qwen3-embedding:8b  | 1
```
→ 임베딩 트래픽 실재. **gemini-embedding-001 = 유료 API** → 비용 가시화 실익 명확(현재 차트에선 0원처럼 안 보임).

### 5.3 임베딩 인덱싱 토큰 적재 (`documents.tokens`)

```
docs=11  with_tokens=11(100%)  total_tokens=444,572  max=126,061
```
→ **인덱싱 임베딩 토큰이 실제로 쌓임**(전 문서 채워짐). 차트 편입 시 즉시 의미 있는 수치(44.5만 토큰).

### 5.4 쿼리타임 RAG (`dataset_queries`)

```
n=496   first=2026-03-11   last=2026-06-08
```
→ 쿼리 RAG 496건 실재하나 **토큰/모델 컬럼 없음**(§2 확정). 496건의 질의 임베딩 토큰은 영영 미기록 → 코어 수정 없이는 차트 불가.

### 5.5 현재 차트가 보여주는 답변 LLM (`messages.model_provider`)

```
yangyaofei/vllm/vllm        : 401
langgenius/ollama/ollama    : 57
langgenius/anthropic/anthropic : 1
```

---

## 6. 차트 편입 난이도 등급 (요약)

| 타입 | 등급 | 사유 |
|---|---|---|
| 답변 LLM / 워크플로·챗플로 LLM 노드 | **Phase 0 완료** | B안으로 이미 차트 진입 |
| **임베딩 인덱싱** | 🟢 **collector 보강(저비용)** | (c)만 빠짐, JOIN 한 줄 + 마트 UNION 한 갈래(~6줄+마이그레이션 1개) |
| 임베딩 쿼리타임 | 🔴 **구조적 불가(코어 수정)** | (b)(c) 결손, `dataset_queries` 컬럼 + `cached_embedding.py` 수정 필요(무수정 원칙 위배) |
| 리랭커 | 🔴 **구조적 불가** | graphon `RerankResult`에 usage 부재(토큰 과금 아님) |
| STT / TTS / moderation | 🔴 **구조적 불가** | invoke 반환에 usage 없음 + 영속 이벤트 없음. 등록 0건 |

---

## 7. 권고안 (phasing)

- **Phase 1 (1순위 권고) — 임베딩 인덱싱 토큰 편입.** 고가치(유료 gemini-embedding 비용 가시화, 44.5만 토큰 실재)·저비용(collector +6줄, 마트 UNION +1갈래, `model_type` 차원). H-DASH-02 잔여 해소의 자연스러운 다음 칸. **단 차트에서 LLM과 분리 표기**(§4.3).
- **Phase 2 (보류 — PM 결정 동반) — 임베딩 쿼리타임.** 496건 실재하나 Dify 코어 수정 + 스키마 컬럼 필요 → **무수정 원칙(아키텍처 불변식 #4) 위배**. 비용 가시화 가치 vs upstream 수정 부담 트레이드오프를 PM이 결정. 권장: 보류.
- **Phase X (불가/제외) — 리랭커·STT·TTS·moderation.** graphon usage 부재로 구조적 불가 + 사내 등록 0건(실익 0). 차트 범위에서 **명시적 제외**로 문서화하고 재론 방지.

---

## 8. PM 결정 필요 항목

1. **차트 커버리지 경계**: "채팅 답변 LLM 비용만" vs "전체 모델 비용(임베딩 인덱싱 포함)". → Phase 1 진행 여부.
2. **혼합 vs 분리 표기**: 임베딩 토큰(1회성 색인 burst)을 LLM 토큰(대화 누적)과 같은 차트/막대에 섞을지, `model_type` 탭·색으로 분리할지(§4.3).
3. **스파이크 노출**: 임베딩 인덱싱은 특정 업로드일에 스파이크(max 12.6만/문서) → 일별 토큰 차트에 그대로 노출 시 표기/주석 방식.
4. **쿼리타임 임베딩(496건)을 위한 Dify 코어 수정 감수 여부** — 무수정 원칙 위배 트레이드오프(Phase 2).
5. **model-call-share에 임베딩 포함 여부** — "호출" 단위 의미 혼선(대화 호출 vs 문서 색인) 고려(§4.4).

---

## 9. 가드 준수 (delegation-standard §1·§2)

- **§1 drift**: 조사 중 design.md ↔ 실제 코드/DB 불일치 **발견 없음**. 단, `data-mart.md §2.4`·§9의 "`mv_model_tokens_daily`는 message_send만 / H-DASH-02 구조적 미가용" 서술은 **B안(20260610 마이그레이션) 적용으로 이미 옛 상태** — 본 조사 범위 밖이나 design 문서 후속 갱신 필요(보고만, 자동 보정 안 함).
- **§2 자가검증**: 본 문서의 모든 "있다/없다"는 위 §1~§5에 **파일:라인 또는 운영 DB SELECT 출력**으로 근거 첨부. graphon usage 부재(rerank/STT/TTS/moderation)는 invoke 반환 타입 코드로, 임베딩 토큰 실재는 운영 `documents.tokens` 집계 출력으로 입증.
- **조사 우선·구현 없음**: 코드/마이그레이션 변경 미실행. §3·§4 diff는 **권고 윤곽**이며 권고안 확정 후 별도 트랙.

## 10. 참조 파일 인덱스 (모두 실검증)

- `graphon/model_runtime/entities/model_entities.py:12-23` — ModelType enum 6종
- `graphon/model_runtime/entities/{text_embedding,rerank,llm}_entities.py` — usage 필드 유무
- `graphon/model_runtime/model_providers/__base/{speech2text,tts,moderation}_model.py` — invoke 반환 타입(usage 없음)
- `api/models/dataset.py:148-149,152,276-284,470,1057-1080` — embedding_model/retrieval_model/documents.tokens/DatasetQuery
- `api/core/indexing_runner.py:581-587,694-697` — 인덱싱 임베딩 토큰 누적 → documents.tokens
- `api/core/rag/embedding/cached_embedding.py:203-207` — 쿼리 임베딩 usage 계산 후 버림
- `api/core/rag/retrieval/dataset_retrieval.py:1042-1053` — dataset_queries INSERT(토큰/모델 없음)
- `api/core/rag/rerank/rerank_model.py:104-106` — invoke_rerank(usage 없음)
- `api/core/workflow/nodes/knowledge_retrieval/knowledge_retrieval_node.py:122-132,212-216` — process_data에 model 미기록
- `dify-audit/src/lib/collectors/documents.ts` — tokens O, 모델식별자 X (보강 대상)
- `dify-audit/src/lib/collectors/{datasets,workflow-nodes}.ts` — datasets는 embedding_model 미SELECT / nodes는 B안 적용됨
- `dify-audit/prisma/audit/migrations/20260610000000_rewrite_model_tokens_daily_union/migration.sql` — 현 UNION 마트
- 운영 DB `192.168.10.159/dify` (읽기전용): provider_models / datasets / documents / dataset_queries / messages 분포
