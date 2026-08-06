---
tags: [프로젝트, dify, AI-Agent, references]
type: references
date: 2026-06-10
external_dependency_observed_at: 2026-06-10
purpose: 모델별 토큰 차트의 모델 분류 메커니즘 검증 + 워크플로우/챗플로우 모델 추출 방안 (A안/B안)
related: [dify-app-modes.md, dify-db-schema.md, audit-details-spec.md, dashboard-query-inventory.md, model-tokens]
---

# 워크플로우/챗플로우 모델 분류 — 소스 검증

> **결론 요약**
> - 모델별 토큰 차트(`spx_mv_model_tokens_daily`)는 `action='message_send'`만 집계 → **모델 정보가 `messages` 행에 있어야 분류됨**.
> - **워크플로우 모델은 "런" 단위가 아니라 "LLM 노드" 단위(`workflow_node_executions.process_data`)에 있고, 현재 수집기가 추출하지 않음** → 모델 차트에서 누락.
> - **챗플로우(advanced-chat)는 message의 모델 컬럼이 `NULL`로 생성됨** → 현재 차트의 `'미분류'` 토큰의 정체가 바로 챗플로우. (워크플로우가 아님)
> - `'미분류'`는 마트 레이어2의 `COALESCE(model_provider,'미분류')`로 생성. **실제 구현되어 있음.**

---

## 0. 배경 질문

> "모델별 차트에서 audit 테이블의 워크플로우가 모델 유형을 못 반환해서 '미분류'로 반환하기로 했다는데, 지금 그렇게 구현돼 있는지 + 워크플로우별 모델 뽑아내려면 어떻게 해야 하는지 조사"

검증 환경: **DB는 목업 데이터 상태**라 토큰 합 대조는 무의미 → **Dify 소스 코드로 직접 검증** (api/ + graphon 패키지 + dify-audit 수집기 + prisma 마이그레이션).

검증 대상 코드베이스: `C:\Users\Administrator\Projects\spx-agent` (`dev` 브랜치)

---

## 1. 모델 분류 데이터 흐름 (전체 파이프라인)

```
[Dify OLTP]                  [수집기 dify-audit]         [audit 테이블]              [마트]                    [백엔드/차트]
messages.model_provider  →  messages.ts (직접 추출)  →  details.modelProvider  →  model_provider_d (gen col) → spx_mv_model_tokens_daily → 모델 차트
messages.model_id            (44-45, 88-89행)            modelId                    model_id_d                   (COALESCE '미분류')

workflow_runs            →  workflow-runs.ts          →  ❌ 모델 키 없음          →  model_*_d = NULL          →  message_send 필터로 제외
                            (모델 컬럼 SELECT 안 함)

workflow_node_executions →  workflow-nodes.ts         →  ❌ node_type만 추출      →  model_*_d = NULL          →  마트 진입 안 함
  .process_data              (process_data 누락)          (model_provider 없음)        (action='workflow_node_execute')
  (model_provider/name)
```

### 핵심 파일·라인

| 단계 | 파일 | 핵심 |
|---|---|---|
| 메시지 수집 | `dify-audit/src/lib/collectors/messages.ts` 44-45, 88-89 | `m.model_provider`, `m.model_id` 직접 SELECT → details에 주입 |
| 워크플로우런 수집 | `dify-audit/src/lib/collectors/workflow-runs.ts` 65-73 | details에 모델 필드 **없음** (totalTokens/appMode만) |
| 노드 수집 | `dify-audit/src/lib/collectors/workflow-nodes.ts` 72-84 | `nodeType`만 추출, `process_data`(=모델) **SELECT 안 함** |
| Generated column | `.../20260515100000_add_generated_columns.../migration.sql` 10-17 | `model_provider_d = details->>'modelProvider'`, `model_id_d`, `total_tokens_d` STORED |
| target_app_id gen col | 동 파일 28-35 | `details ? 'appId'`이면 appId 사용 → 노드 이벤트도 app 매핑 가능 |
| Layer1 enriched | `.../20260515200000_add_mart_layer1_views/migration.sql` 54 | `WHERE action IN ('message_send','workflow_execute')` — 노드 액션 제외 |
| Layer2 모델토큰 | `.../20260515300000_add_mart_layer2_daily_mviews/migration.sql` 54-59 | `COALESCE(model_provider,'미분류')` + `WHERE action='message_send'` |
| 백엔드 서비스 | `api/services/admin/dashboard_model_tokens_service.py` 55-60 | `provider or 'unknown'`, `LOCAL_PROVIDERS` 로컬 판별, `(로컬)` 접미사 |
| 호출점유 서비스 | `api/services/admin/dashboard_drill_calls_service.py` (model-call-share) | enriched view에서 `model_id IS NOT NULL` 필터 → NULL 모델 제외 |

---

## 2. '미분류'(COALESCE) — 실제 구현 확인됨

`spx_mv_model_tokens_daily` (Layer2, migration `20260515300000` 50-60행):

```sql
CREATE MATERIALIZED VIEW public.spx_mv_model_tokens_daily AS
SELECT
  tenant_id,
  date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul') AS day,
  COALESCE(model_provider,'미분류') AS model_provider,   -- ← '미분류' 치환 (구현됨)
  COALESCE(model_id,'미분류')       AS model_id,
  SUM(total_tokens) AS tokens, COUNT(*) AS calls
FROM public.spx_mv_audit_enriched   -- ⚠️ 5/20 rename: 옛 spx_v_audit_enriched → 현 spx_mv_audit_enriched
WHERE NOT is_debug AND action='message_send' AND total_tokens IS NOT NULL  -- ← 워크플로우 제외
GROUP BY 1,2,3,4
```

- `COALESCE(...,'미분류')` → **모델이 NULL이면 '미분류'로 치환. 실제 존재함.**
- 단, `WHERE action='message_send'` → **워크플로우(`workflow_execute`)는 통째로 제외.** 워크플로우는 '미분류'로도 안 잡히고 그냥 빠짐.
- 백엔드에서 추가 fallback: `provider = row[0] or "unknown"` (남은 NULL 방어).

---

## 3. 워크플로우가 모델을 못 반환하는 이유

모델 정보는 **워크플로우 런이 아니라 LLM 노드 실행 단위**에 저장됨.

- `workflow_runs` 테이블/수집기에 모델 컬럼 자체가 없음.
- 모델을 기록하는 곳: **모델 보유 노드 3종의 `process_data`** (graphon 패키지):

| node_type (문자열) | 파일 | process_data 모델 키 |
|---|---|---|
| `llm` | `graphon/nodes/llm/node.py` 309-318 | `model_provider`, `model_name`, `usage` |
| `parameter-extractor` | `graphon/nodes/parameter_extractor/parameter_extractor_node.py` 226-248 | `model_provider`, `model_name`, `usage` |
| `question-classifier` | `graphon/nodes/question_classifier/question_classifier_node.py` 237-245 | `model_provider`, `model_name`, `usage` |

> node_type 문자열 상수: `graphon/enums.py` — `LLM="llm"`, `QUESTION_CLASSIFIER="question-classifier"`, `PARAMETER_EXTRACTOR="parameter-extractor"`

- `workflow-nodes.ts` 수집기는 `node_type`만 details에 넣고 `process_data`는 SELECT조차 안 함 → 모델 영영 누락.
- ⚠️ `process_data`는 프롬프트가 크면 외부 스토리지로 오프로딩될 수 있음 (`api/models/workflow.py`의 `process_data_truncated` / `load_full_process_data`). 잘린 경우 인라인에 모델이 없을 수 있음 → 추가 조회 필요할 수 있음.

---

## 4. ⚠️ 핵심 발견 — 챗플로우(advanced-chat) message 모델 = NULL

소스 검증으로 드러난 가장 중요한 사실.

### 4-1. 챗플로우 message 토큰 = 워크플로우 전체 노드 합

- `api/core/app/apps/advanced_chat/generate_task_pipeline.py` 958-963:
  ```python
  if graph_runtime_state and graph_runtime_state.llm_usage:
      usage = graph_runtime_state.llm_usage
      message.message_tokens = usage.prompt_tokens
      message.answer_tokens = usage.completion_tokens
  ```
- `graph_runtime_state.llm_usage`는 **모든 모델 노드의 usage를 누적한 값**:
  - `graphon/nodes/base/usage_tracking_mixin.py` `_accumulate_usage`: `current_usage.plus(usage)`
  - `graphon/graph_engine/event_management/event_handlers.py` 197/270/297: `_accumulate_node_usage(event.node_run_result.llm_usage)`
- → **챗플로우 message.total_tokens = 그 챗플로우 안 모든 노드 토큰의 합.** (이중 집계 위험 확정)

### 4-2. 챗플로우 message 모델 컬럼은 NULL로 생성됨

- `api/core/app/apps/message_based_app_generator.py` 140-144:
  ```python
  if isinstance(application_generate_entity, AdvancedChatAppGenerateEntity):
      app_model_config_id = None
      override_model_configs = None
      model_provider = None   # ← 챗플로우는
      model_id = None         # ← 모델을 message에 안 박음
  ```
- 이후 `Message(model_provider=None, model_id=None, ...)` (192-195행)로 생성.
- → **챗플로우 메시지는 항상 `model_provider/model_id = NULL`** → Layer2에서 `COALESCE(...,'미분류')` 적용 → **'미분류' 버킷으로 들어감.**

---

## 5. 현재 상태 — 모델 차트 3-버킷 정리

| 앱 모드 | message 모델 | 현재 모델 차트에서 | 토큰 위치 |
|---|---|---|---|
| chat / agent-chat / completion | 실제 모델 채워짐 | **정상 분류** | message에 모델+토큰 |
| **advanced-chat (챗플로우)** | **NULL** | **'미분류'로 집계** | message에 토큰(노드 합), 모델은 NULL |
| **workflow (순수)** | message 자체 없음 | **누락 (집계 안 됨)** | 노드 process_data에만 토큰/모델 |

> **즉, 현재 차트의 '미분류' = 워크플로우가 아니라 챗플로우(advanced-chat).** 순수 워크플로우는 미분류로도 안 잡히고 빠져 있음.

---

## 6. 모델 추출 방안 — A안 / B안

방향: **노드 단위** 추출 (런 단위 롤업은 멀티모델 처리 복잡 → 보류).

### 공통 작업

1. **수집기** (`workflow-nodes.ts`): SQL에 `n.process_data` 추가 → `node_type IN ('llm','question-classifier','parameter-extractor')`일 때 파싱:
   ```ts
   const pd = r.process_data ? JSON.parse(r.process_data) : {}
   details: { ...,
     modelProvider: pd.model_provider ?? null,
     modelId: pd.model_name ?? null,
     totalTokens: pd.usage?.total_tokens ?? null,
   }
   ```
   → **Generated column이 키만 맞으면 자동 추출(`model_provider_d` 등). 추출용 새 마이그레이션 불필요.** ✅
2. **전용 얇은 mview** 신설 (공유 `spx_mv_audit_enriched`를 넓히지 말 것 — 다른 차트 bloat 방지):
   ```sql
   CREATE MATERIALIZED VIEW public.spx_mv_workflow_model_tokens_daily AS
   SELECT tenant_id,
          date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul') AS day,
          COALESCE(model_provider_d,'미분류') AS model_provider,
          COALESCE(model_id_d,'미분류')       AS model_id,
          SUM(total_tokens_d) AS tokens, COUNT(*) AS calls
   FROM public.spx_audit_events
   WHERE action = 'workflow_node_execute'
     AND app_mode_d = 'workflow'      -- A안: 순수 workflow만. B안은 §6 UNION 형태로 확장
     AND model_id_d IS NOT NULL
   GROUP BY 1,2,3,4;
   ```
3. **부분 인덱스** + `REFRESH MATERIALIZED VIEW CONCURRENTLY` (기존 `spx_audit_events_action_mart_idx` 패턴).

> **왜 `spx_audit_events` 직접 조회인가 (enriched 우회 — 설계 근거)**:
> - enriched는 `WHERE action IN ('message_send','workflow_execute')`라 **`workflow_node_execute`(노드 행)가 없음** → enriched로는 노드 모델을 못 봄. 넣으려면 공유 enriched를 넓혀야 하는데 그러면 kpi_calls·drill 등 **6개 차트가 스캔하는 뷰가 노드 행으로 부풀음**.
> - 모델 차트는 **부서 차원이 없어** enriched의 RBAC 조인이 불필요하고, **챗플로우를 노드로 봐야** 해서 enriched의 `is_canonical_call`(message_send 유지)과는 오히려 반대 → enriched 경유가 부적합.
> - **유일한 비용 = is_debug 중복**: 직접 조회라 디버그 필터(`invoke_from_d='debugger' OR triggered_from_d IN ('debugging','rag-pipeline-debugging')`)를 인라인 복제. enriched의 `is_debug`와 **동일하게 유지 의무** (drift 감시). enriched is_debug 변경 시 model_tokens 마이그레이션도 동반 갱신.

### A안 — 순수 workflow만 (가벼움, ~1~2일)

- 노드 채널에 `app_mode_d='workflow'`만 포함.
- 챗플로우는 기존 message_send(='미분류') 그대로 둠.
- **결과: 순수 워크플로우는 차트에 등장, 챗플로우 '미분류'는 그대로 남음.**

### B안 — workflow + advanced-chat (미분류 해소, ~3~4일) ← 선호

- 챗플로우도 노드 채널로 추출 → 실제 모델별 재분류.
- **범위 = 모델 차원 차트 2종**: ① 모델별 토큰(`dashboard_model_tokens_service.py`) ② 모델별 호출 점유(`drill_calls.get_model_call_share`). 둘 다 같은 노드 모델 마트에서 공급(`tokens`/`calls` 컬럼). 그 외 부서/앱 차원 차트는 손대지 않음 (§6-1).
- 정확도 근거: 챗플로우 message는 모델 NULL + 여러 모델 토큰의 합이라, **메시지 레벨에선 모델별 분해 원천 불가 → 노드 레벨이 유일한 방법.**

**B안 model_tokens mview 최종 형태 (UNION) — 챗플로우를 (1)에서 빼고 (2)에서 재분류:**
```sql
-- (1) 비-챗플로우 메시지: 기존 모델 그대로
SELECT ... FROM public.spx_audit_events
WHERE action='message_send' AND app_mode_d <> 'advanced-chat' AND model_id_d IS NOT NULL
UNION ALL
-- (2) 워크플로우 + 챗플로우 노드: 노드 process_data에서 모델
SELECT ... FROM public.spx_audit_events
WHERE action='workflow_node_execute'
  AND app_mode_d IN ('workflow','advanced-chat') AND model_id_d IS NOT NULL
```
→ 챗플로우가 (1)에서 제외되고 (2)에서 실제 모델로 분해 → 이중집계 없음 + '미분류' 해소. 총 토큰은 보존(§4-1: message 합 = 노드 합).

---

## 6-1. ⚠️ 교차 의존성 — B안은 model_tokens 한 곳에만 (필독)

> "챗플로우를 노드에서 읽는 건 **모델 분류 차트에만** 한정해야 하는가?" → **예. 호출/토큰 집계는 절대 노드로 바꾸면 안 됨.**

대시보드 거의 전 차트가 `spx_mv_kpi_calls_daily`(또는 enriched raw)를 쓰고, 그 집계 게이트가 Layer1의 **`is_canonical_call`** (migration `20260515200000` 41-44행):
```sql
COALESCE(ae.app_mode_d <> 'advanced-chat' OR ae.action = 'message_send', FALSE) AS is_canonical_call
```
- **advanced-chat → `message_send` 행만 canonical** (workflow_execute 제외) ← **H-DASH-01의 실제 구현체**
- 순수 workflow → workflow_execute / chat·completion → message_send
- `spx_mv_kpi_calls_daily`는 호출 수 + **토큰까지** `SUM(total_tokens)` 함 (migration `20260515300000` 25행)

### message_send(챗플로우)를 쓰는 차트 전수

> ⚠️ **차원 구분 주의**: 차트는 두 종류다.
> - **부서/앱 차원(canonical 호출)** — message_send 유지, **B안 무관**
> - **모델 차원(모델 귀속)** — 노드 기반으로 가야 정확, **B안 대상**
> `dashboard_drill_calls_service.py`는 한 파일에 두 차원이 **섞여 있음**(아래 표 주의).

| 차트/서비스 | 차원 | 소스 | B안 |
|---|---|---|---|
| KPI 카드 (호출량/이용앱/인기앱) `dashboard_kpi_service.py` | 부서/앱 | `spx_mv_kpi_calls_daily` | 유지 (message_send) |
| 부서별 활동 (호출·토큰·에러) `dashboard_dept_activity_service.py` | 부서 | `spx_mv_kpi_calls_daily` | 유지 |
| 부서별 호출 수/RPS `drill_calls.get_dept_call_count` / `_rps` | 부서 | `spx_mv_kpi_calls_daily` | 유지 |
| 에러 드릴 `dashboard_drill_errors_service.py` | 부서/앱 | kpi_calls + enriched | 유지 |
| 앱/유저 드릴 `dashboard_drill_apps/users_service.py` | 앱/유저 | `spx_mv_audit_enriched` raw | 유지 |
| **모델별 토큰 `dashboard_model_tokens_service.py`** | **모델** | `spx_mv_model_tokens_daily` | **노드로 교체** |
| **모델별 호출 점유 `drill_calls.get_model_call_share`** (48-66행) | **모델** | enriched 직접 (`model_id IS NOT NULL`) | **노드로 교체** |

> **모델별 호출 점유도 model_tokens와 같은 사각지대**: enriched에서 `model_id IS NOT NULL`만 세므로 챗플로우(모델 NULL)·워크플로우(모델 NULL) 모두 누락. 현재는 비-챗플로우 대화앱만 반영됨.
> → **모델 차원 차트 2종은 동일한 노드 기반 모집단을 공유해야** 서로 정합. 실무적으로 `get_model_call_share`를 노드 모델 마트의 `COUNT(*) AS calls` 컬럼으로 **repoint**하면 됨 (마트에 `tokens`/`calls` 둘 다 있음). 두 차트 한 마트에서 공급.

### ⚠️ 모델 호출 수 ≠ canonical 호출량 (UI 표기 필수)

- **KPI 총 호출량** = 사용자 요청 수 (canonical, 워크플로우 1런 = 1건)
- **모델별 호출 점유** = 모델 호출 수 (노드 단위, 워크플로우 1런 = LLM 노드 N건)
- → **모델별 호출 점유 합 ≥ 총 호출량**이 정상. "왜 합이 안 맞나" 오해 방지 위해 툴팁/문서에 단위 차이 명시.

### 호출 수를 노드로 바꾸면 안 되는 이유 (부서/앱 차원)

챗플로우 **1턴 = message_send 1건 = canonical call 1건**인데, 노드로 세면 1턴이 노드 N개로 잡혀 **호출 수가 N배 부풀려짐.** KPI·부서활동·드릴 전부 깨짐. → 호출/토큰 집계는 message_send 유지, **모델 *라벨*만 model_tokens에서 노드로** 가져온다.

### 토큰 일관성 (검증 포인트)

- §4-1에 의해 챗플로우 `message.total_tokens = 노드 토큰 합` → `model_tokens`(노드) 총합 == `kpi_calls`(message) 챗플로우 총합 **일치해야 정상.**
- ⚠️ `process_data` 오프로딩으로 일부 노드 모델/토큰 누락 시 → model_tokens 합 < kpi_calls 합 **불일치** 발생 가능 (차트 간 총합 어긋남 리스크).

---

## 7. H-DASH-01 불변식 — 수정 범위 (B안 진행 시)

프로젝트 아키텍처 불변식 #1 / `hdd/defect-catalog.md` H-DASH-01:
> **"ADVANCED_CHAT은 `messages`만 읽음 — `workflow_runs` 이중카운트 금지"**

- **SoT 통째 전환 불필요** (§6-1 확인). 호출/토큰 집계(`is_canonical_call` → kpi_calls)는 **그대로 message_send 유지.**
- 좁은 예외만 추가: **"모델 차원 차트 2종(모델별 토큰 + 모델별 호출 점유)에 한해, 챗플로우/워크플로우의 모델 *라벨*은 노드 `process_data`에서 가져온다. 단 총 토큰은 message 합과 일치해야 한다."**
- 이중카운트 금지 원칙은 그대로 — 모델 마트 안에서 챗플로우를 message_send/노드 양쪽으로 세지 않게 (§6 UNION의 `app_mode_d <> 'advanced-chat'` 분기로 보장).
- defect-catalog는 작성자(본인) 수정 가능 → H-DASH-01에 "단, 모델 차원 차트의 모델 라벨/호출수는 노드 출처 예외" 한 줄 추가하는 방향.

---

## 8. 리스크 체크리스트

- [ ] **B안 범위 한정** — 노드 교체는 **모델 차원 차트 2종**(model_tokens + model-call-share)만. 부서/앱 차원(kpi_calls·enriched)은 message_send 유지 (§6-1). canonical 호출 수를 노드로 세면 N배 부풀려짐.
- [ ] **모델 호출 수 단위 표기** — 모델별 호출 점유(노드 단위) 합 ≥ KPI 총 호출량(canonical). UI 툴팁/문서에 단위 차이 명시 (§6-1).
- [ ] **모델 차트 2종 정합** — model_tokens(tokens)·model-call-share(calls)를 같은 노드 마트에서 공급해 모집단 일치.
- [ ] **토큰 이중 집계** — 모델 마트 안에서 챗플로우를 message_send와 노드 양쪽에서 세지 않도록 `app_mode_d <> 'advanced-chat'` 분기 (§6 UNION). 검증: 합계 대조.
- [ ] **차트 간 총합 일치** — model_tokens(노드) 챗플로우 합 == kpi_calls(message) 챗플로우 합 (§6-1). 오프로딩 시 어긋남 감시.
- [ ] **process_data 오프로딩** — 큰 프롬프트 시 외부 스토리지로 빠져 인라인 모델 누락 가능 (`process_data_truncated`). 일부 노드 '미분류' 잔존 가능.
- [ ] **마트 볼륨** — 노드 행은 **이미 audit 테이블에 적재 중**(workflow-nodes 수집기 가동). 모델 보유 3종만 필터하면 대상 축소. 일별 집계 출력은 작음(모델×일×테넌트). 비용은 REFRESH 스캔 → 부분 인덱스로 완화.
- [ ] **백필** — 기존 노드 행엔 모델 키 없음 → 재수집(커서 리셋) 또는 details UPDATE 필요(STORED gen col 재계산).
- [ ] **H-DASH-01 예외 추가** — B안 선행. SoT 전환 아님, "model_tokens 모델 라벨만 노드 출처" 좁은 예외 (§7).

---

## 9. 참조 파일 인덱스 (모두 실검증)

- `dify-audit/src/lib/collectors/messages.ts` — 메시지 모델 직접 추출
- `dify-audit/src/lib/collectors/workflow-runs.ts` — 모델 미추출
- `dify-audit/src/lib/collectors/workflow-nodes.ts` — node_type만, process_data 누락
- `dify-audit/prisma/audit/migrations/20260515100000_.../migration.sql` — generated columns
- `dify-audit/prisma/audit/migrations/20260515200000_.../migration.sql` — Layer1 enriched (action 필터)
- `dify-audit/prisma/audit/migrations/20260515300000_.../migration.sql` — Layer2 '미분류' COALESCE
- `api/services/admin/dashboard_model_tokens_service.py` — 모델/로컬 분류, unknown fallback (B안 교체 대상 ①)
- `api/services/admin/dashboard_drill_calls_service.py` — `get_model_call_share` 48-66 (모델 차원, `model_id IS NOT NULL`, B안 교체 대상 ②) + `get_dept_call_count`/`_rps` (부서 차원, 유지)
- `api/services/admin/dashboard_kpi_service.py` / `dashboard_dept_activity_service.py` / `dashboard_drill_{apps,users,errors}_service.py` — kpi_calls·enriched 경유, 챗플로우=message_send (§6-1, B안에서 손대지 않음)
- migration `20260515200000` 41-44 — `is_canonical_call` 정의 (H-DASH-01 구현체)
- migration `20260515300000` 15-30 — `spx_mv_kpi_calls_daily` (호출·토큰 집계, message_send 기반)
- `api/core/app/apps/advanced_chat/generate_task_pipeline.py` 958-963 — 챗플로우 message 토큰 = 노드 합
- `api/core/app/apps/message_based_app_generator.py` 140-144 — 챗플로우 message 모델 = NULL
- `api/.venv/.../graphon/nodes/{llm,parameter_extractor,question_classifier}/*.py` — process_data 모델 기록
- `api/.venv/.../graphon/nodes/base/usage_tracking_mixin.py` + `graph_engine/event_management/event_handlers.py` — usage 누적
- `api/.venv/.../graphon/enums.py` — node_type 문자열 상수
