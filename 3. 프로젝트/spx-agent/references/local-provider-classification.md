---
tags: [프로젝트, dify, AI-Agent, references]
type: references
date: 2026-06-10
purpose: 로컬 모델 판별 기준 보강 조사 — 신호 비교·DB 실증·권고안
related: [workflow-model-classification.md, audit-details-spec.md, dify-db-schema.md]
verified_from: api/models/provider.py, api/services/admin/dashboard_model_tokens_service.py, dify-audit/ 코드, 192.168.10.159 DB 직접 조회
---

# 로컬 모델 판별 기준 분석

## 변경 이력

- 2026-06-10: 초안 — 조사 1~5 전항 완료, 신호 비교표 + 권고안 + PM 결정 필요 항목

---

## 0. 현재 상태 (한계)

```python
# api/services/admin/dashboard_model_tokens_service.py
LOCAL_PROVIDERS = frozenset({"ollama", "xinference", "localai"})
```

- "(로컬)" 라벨 = `provider in LOCAL_PROVIDERS`일 때만 붙음.
- **사용처 2곳**: `dashboard_model_tokens_service.py`, `dashboard_drill_calls_service.py::get_model_call_share`.
- **조사 동기**: vLLM이 `openai_api_compatible` provider로 등록될 수 있다는 가정 → 이름만으로는 로컬/원격 구분 불가한 케이스가 존재할 수 있음. (PROMPT.md 전제 — 실제 환경별로 다를 수 있으며, 조사 4에서 확인)

---

## 1. 조사 1 — 엔드포인트 URL 신호

### 1.1 저장 위치

> **주의**: 이 코드베이스에서는 마이그레이션 `2025_08_13_1605`으로 `providers`/`provider_models` 테이블의 `encrypted_config` 컬럼이 **제거**되고, 별도 credential 테이블로 이관됨.

| 테이블 | 컬럼 | 내용 | 근거 |
|--------|------|------|------|
| `provider_credentials` | `encrypted_config` (LongText) | provider 레벨 credential (api_key, base_url 등) | `api/models/provider.py:303-327` |
| `provider_model_credentials` | `encrypted_config` (LongText) | 개별 모델 레벨 credential | `api/models/provider.py:329-361` |
| `load_balancing_model_configs` | `encrypted_config` (LongText) | LB별 credential | `api/models/provider.py:270-301` |
| ~~`providers`~~ | ~~`encrypted_config`~~ | **제거됨** → `credential_id` FK로 교체 | 마이그레이션 `2025_08_13_1605` |
| ~~`provider_models`~~ | ~~`encrypted_config`~~ | **제거됨** → `credential_id` FK로 교체 | 마이그레이션 `2025_08_13_1605` |

**endpoint URL은 별도 컬럼 없음** — base_url은 `encrypted_config` JSON 내부에 provider별 다른 키로 저장:
- OpenAI: `openai_api_base` (`api/core/hosting_configuration.py:151`)
- Azure OpenAI: `openai_api_base` (`:72`)
- Anthropic: `anthropic_api_url` (`:214`)
- Deepseek: `endpoint_url` (`:302`)

### 1.2 암호화 여부

**암호화됨** — 테넌트별 **RSA 공개키**로 암호화 (`api/core/helper/encrypter.py:18-26`). SQL `SELECT`만으로는 읽을 수 없음.
- 암호화: `rsa.encrypt()` + `tenant.encrypt_public_key` → base64 인코딩
- 복호화: `rsa.decrypt()` + 테넌트 개인키 (`encrypter.py:29-30`)
- Flask(Python) 런타임에서 `ProviderManager._get_and_decrypt_credentials()` (`provider_manager.py:851-899`)로 복호화.
- dify-audit collector(TypeScript)에서는 복호화 불가 (RSA 키 접근 + 별도 구현 필요).

### 1.3 판별 가능성

복호화 시 base_url에서:
- private IP (`10.x`, `172.16-31.x`, `192.168.x`) / `localhost` / `127.0.0.1` → **로컬**
- `api.openai.com`, `api.anthropic.com` 등 공개 도메인 → **클라우드**

### 1.4 치명적 제약

- audit/mart 데이터에 endpoint **없음** — mart에는 `provider`(이름 문자열) + `model`만 존재.
- 사용하려면 credential 테이블(`provider_credentials` / `provider_model_credentials`)과 JOIN + 복호화 필요 → 매 대시보드 조회마다 비용 발생.

### 1.5 결론

| 항목 | 평가 |
|------|------|
| 정확도 | **최고** (private IP 기반 판별) |
| 데이터 가용성 | **낮음** (암호화 + mart에 없음) |
| 구현 비용 | **높음** (복호화 로직 + 캐시 필요) |

---

## 2. 조사 2 — provider_type 신호

### 2.1 DB 컬럼 확인

`providers.provider_type` 존재 — 그러나 값은 **`custom`/`system`** (자격증명 범위 구분).

```
-- 운영 DB 스냅샷 (192.168.10.159, 참고용)
 provider_name                                  | provider_type
-------------------------------------------------+--------------
 langgenius/anthropic/anthropic                  | custom
 langgenius/gemini/google                        | custom
```

> 이 DB에서는 provider-level credential이 있는 2건만 `providers` 테이블에 존재. ollama/vllm 등은 model-level credential(`provider_model_credentials`)만 사용.

- `ProviderType`은 `CUSTOM`/`SYSTEM`(자격증명 범위)이며 로컬/클라우드 구분이 아님.
- 코드 정의: `api/models/provider.py:21-30` — `CUSTOM`은 사용자 제공 credential, `SYSTEM`은 시스템 기본 제공.
- 클라우드 provider도 `CUSTOM`이 될 수 있음 → **로컬 판별 신호로 사용 불가**.

### 2.2 configurate_method (predefined vs customizable)

- `ConfigurateMethod` enum 정의: `api/.venv/Lib/site-packages/graphon/model_runtime/entities/provider_entities.py:10-16`
  ```python
  class ConfigurateMethod(StrEnum):
      PREDEFINED_MODEL = "predefined-model"
      CUSTOMIZABLE_MODEL = "customizable-model"
  ```

- **`dify_plugin` DB에서 SQL 질의 가능** (초기 조사 "질의 불가" 결론 정정):
  - 테이블: `dify_plugin.plugin_declarations.declaration` (JSON Text)
  - 경로: `declaration -> 'model' -> 'configurate_methods'`

  | plugin_id | model.provider | configurate_methods |
  |-----------|---------------|---------------------|
  | `langgenius/ollama` | ollama | `[customizable-model]` |
  | `yangyaofei/vllm` | vllm | `[customizable-model]` |
  | `langgenius/openai_api_compatible` | openai_api_compatible | `[customizable-model]` |
  | `langgenius/openai` | openai | `[predefined-model, customizable-model]` |
  | `langgenius/anthropic` | anthropic | `[predefined-model, customizable-model]` |
  | `langgenius/gemini` | google | `[predefined-model]` |

- `customizable-model` only = 사용자가 endpoint를 직접 설정하는 provider (ollama, vllm, openai_api_compatible)
- `predefined-model` 포함 = 클라우드 API 키만으로 사용 가능 (anthropic, openai, gemini)
- **단, `openai_api_compatible`도 `customizable-model`** → customizable = 로컬이라는 등식은 성립하지 않음

### 2.3 결론

| 항목 | 평가 |
|------|------|
| `provider_type` (DB) 정확도 | **낮음** (`custom`/`system`은 자격증명 범위이며 로컬/클라우드 구분 아님) |
| `configurate_method` 정확도 | **중간** (customizable ≈ 자체호스팅이지만 openai_api_compatible도 포함) |
| 데이터 가용성 | `provider_type` → dify DB 즉시 / `configurate_method` → **dify_plugin DB에서 JSON 추출 가능** |
| **판정**: 단독 사용 불가 (보조 신호로는 활용 가능) | |

---

## 3. 조사 3 — 플러그인 메타데이터

### 3.1 provider 디렉토리 상태

`api/core/model_runtime/model_providers/` → **비어있음** (플러그인 기반 아키텍처 확인).
- provider 정의는 플러그인 데몬에서 동적 로드 (`api/core/plugin/entities/plugin_daemon.py:86-94`).
- `_position.yaml` (`graphon/model_runtime/model_providers/_position.yaml`)에 provider 로드 순서만 존재.

### 3.2 로컬/클라우드 플래그

**명시적 플래그 없음** — `is_local`, `deployment_type`, `category` 등 직접적인 로컬/클라우드 분류 필드는 없음.

### 3.3 간접 신호 — credential schema의 endpoint 필드 (`dify_plugin` DB에서 확인)

`dify_plugin.plugin_declarations.declaration` JSON의 `model.model_credential_schema.credential_form_schemas`에서 endpoint 관련 필드 패턴:

| provider | endpoint 변수명 | 필수 여부 | 비고 |
|----------|----------------|-----------|------|
| ollama | `base_url` | **required** | 자체호스팅 전용 |
| vllm | `endpoint_url` | **required** | 자체호스팅 전용 |
| openai_api_compatible | `endpoint_url` | **required** | 로컬/원격 모두 가능 |
| anthropic | `anthropic_api_url` | optional | 클라우드 기본, 커스텀 endpoint 선택 |
| gemini | (없음) | - | 클라우드 전용 |

- `customizable-model` only provider는 endpoint가 **required** → 자체호스팅 가능성 높음
- `predefined-model` only provider(gemini)는 endpoint 필드 자체가 없음
- **단, endpoint required ≠ 로컬**: `openai_api_compatible`은 required이지만 원격 서비스도 가능

### 3.4 결론

| 항목 | 평가 |
|------|------|
| 명시적 로컬 플래그 | **없음** |
| 간접 신호 (configurate_methods + endpoint required) | `dify_plugin` DB에서 **SQL 질의 가능** (§ 2.2 참조) |
| 정확도 | 중간 (customizable + endpoint required ≈ 자체호스팅, 단 openai_api_compatible 예외) |
| 구현 비용 | 중간 (dify_plugin DB JSON 파싱 필요, 단 dify DB와 별도 DB) |
| **판정**: 보조 신호로 활용 가능, 단독 판별에는 부족 | |

---

## 4. 조사 4 — 실제 워크스페이스 provider 현황

> 소스: `192.168.10.159:5432/dify` **psql 직접 조회** (2026-06-10)

### 4.0 주요 발견 — provider 이름이 플러그인 경로 형식

DB 내 `provider_name`은 짧은 이름(`ollama`)이 아니라 **플러그인 경로 형식**:

```
{author}/{plugin_name}/{provider_name}
```

예: `langgenius/ollama/ollama`, `yangyaofei/vllm/vllm`

> **⚠️ 현재 `LOCAL_PROVIDERS = {"ollama", "xinference", "localai"}`는 짧은 이름** — DB/mart의 플러그인 경로와 **매칭 안 될 가능성**. 대시보드 서비스가 mart의 `model_provider` 값과 비교할 때 형식 불일치 확인 필요.

### 4.1 등록된 provider (`providers` 테이블)

| provider_name (플러그인 경로) | provider_type | is_valid | credential_id |
|-------------------------------|--------------|----------|---------------|
| `langgenius/anthropic/anthropic` | custom | true | 있음 |
| `langgenius/gemini/google` | custom | true | 있음 |

> **참고**: `providers` 테이블에는 provider-level credential이 있는 2건만 등록. ollama/vllm/openai_api_compatible은 **model-level credential**만 있어 `providers` 테이블에는 행 없음 → `provider_model_credentials` 테이블에서 확인.

### 4.2 등록된 모델 (`provider_models` 테이블)

> "성격" 컬럼은 DB 데이터가 아닌 provider 이름 기반 추정. endpoint 복호화 없이는 실제 로컬/원격 확인 불가.

| provider_name | model_name | model_type |
|---------------|-----------|------------|
| `langgenius/ollama/ollama` | gpt-oss:latest | text-generation |
| `langgenius/ollama/ollama` | qwen3:30b | text-generation |
| `langgenius/ollama/ollama` | qwen3-vl:32b | text-generation |
| `langgenius/ollama/ollama` | bge-m3 | embeddings |
| `langgenius/ollama/ollama` | qwen3-embedding:8b | embeddings |
| `yangyaofei/vllm/vllm` | /models/gpt-oss-20b | text-generation |
| `yangyaofei/vllm/vllm` | gemma-4-26b | text-generation |
| `yangyaofei/vllm/vllm` | gpt-oss-120b | text-generation |
| `yangyaofei/vllm/vllm` | gpt-oss-120b-test | text-generation |
| `yangyaofei/vllm/vllm` | /models/en3-Embedding-0.6B | text-generation |
| `yangyaofei/vllm/vllm` | model/bge-reranker-v2-m3 | text-generation |
| `langgenius/openai/openai` | bge-m3-ko | embeddings |
| `langgenius/openai_api_compatible/openai_api_compatible` | bge-m3-ko | embeddings |

### 4.3 실제 워크플로우 호출 현황 (`workflow_node_executions.process_data`)

> ⚠️ 이 데이터는 **조사 시점의 운영 DB 스냅샷**이며, provider 구성·사용량은 수시로 변동됨. 특정 provider 비중으로 우선순위를 판단하지 말 것.

| model_provider (workflow_node_executions 기준) | model_name | 호출 수 |
|------------------------------------------------|-----------|---------|
| `yangyaofei/vllm/vllm` | gpt-oss-120b, gemma-4-26b, /models/gpt-oss-20b | 1,113 |
| `langgenius/anthropic/anthropic` | claude-sonnet-4-6 | 157 |
| `langgenius/ollama/ollama` | gpt-oss:latest, qwen3:30b | 123 |

### 4.4 credential 정보

| 위치 | provider_name | credential_name | config_len |
|------|---------------|-----------------|-----------|
| `provider_credentials` | `langgenius/anthropic/anthropic` | claude | 565 |
| `provider_credentials` | `langgenius/gemini/google` | Free Tier | 470 |
| `provider_model_credentials` | `langgenius/ollama/ollama` | all / API KEY 1~2 | 66~166 |
| `provider_model_credentials` | `yangyaofei/vllm/vllm` | all | 418~498 |
| `provider_model_credentials` | `langgenius/openai/openai` | API KEY 1~2 | 494 |
| `provider_model_credentials` | `langgenius/openai_api_compatible/...` | all | 219 |

- `encrypted_config`는 RSA 암호화 — 복호화 없이 endpoint 확인 불가.
- `load_balancing_model_configs` 1건 (anthropic/claude-sonnet-4-6, `__inherit__`).

### 4.5 기본 모델 설정 (`tenant_default_models`)

| model_type | provider | model |
|-----------|---------|-------|
| text-generation | `yangyaofei/vllm/vllm` | gemma-4-26b |
| text-embedding | `langgenius/ollama/ollama` | qwen3-embedding:8b |
| embeddings | `langgenius/openai_api_compatible/...` | bge-m3-ko |

### 4.6 `messages.model_provider` 형식 확인

`messages` 테이블도 `model_provider` 컬럼(varchar)에 플러그인 경로 형식으로 저장:

```
langgenius/anthropic/anthropic    →  claude-sonnet-4-6
langgenius/ollama/ollama          →  gpt-oss:latest, qwen3:30b
yangyaofei/vllm/vllm              →  gpt-oss-120b, gemma-4-26b, ...
```

`workflow_node_executions.process_data`(JSON)와 `messages.model_provider`(컬럼) 두 경로 모두 동일한 플러그인 경로 형식. mart의 `model_provider`도 같은 형식일 가능성 높음.

### 4.7 provider 이름 형식 — 테이블별 차이 (`dify_plugin` DB 보충)

`dify_plugin.ai_model_installations` 테이블에서는 `provider` 컬럼이 **짧은 이름**으로 저장:

| provider (짧은 이름) | plugin_id (플러그인 경로) |
|----------------------|--------------------------|
| `anthropic` | `langgenius/anthropic` |
| `google` | `langgenius/gemini` |
| `ollama` | `langgenius/ollama` |
| `openai` | `langgenius/openai` |
| `openai_api_compatible` | `langgenius/openai_api_compatible` |
| `vllm` | `yangyaofei/vllm` |

**테이블별 provider 이름 형식 정리:**

| 테이블 (DB) | 컬럼 | 형식 | 예시 |
|-------------|------|------|------|
| `providers` (dify) | `provider_name` | 플러그인 경로 (3세그먼트) | `langgenius/ollama/ollama` |
| `provider_models` (dify) | `provider_name` | 플러그인 경로 (3세그먼트) | `yangyaofei/vllm/vllm` |
| `provider_model_credentials` (dify) | `provider_name` | 플러그인 경로 (3세그먼트) | `langgenius/ollama/ollama` |
| `messages` (dify) | `model_provider` | 플러그인 경로 (3세그먼트) | `yangyaofei/vllm/vllm` |
| `workflow_node_executions` (dify) | `process_data` JSON | 플러그인 경로 (3세그먼트) | `langgenius/anthropic/anthropic` |
| `ai_model_installations` (dify_plugin) | `provider` | **짧은 이름** | `ollama` |
| `plugin_declarations` (dify_plugin) | `declaration` JSON → `model.provider` | **짧은 이름** | `vllm` |

> dify DB는 3세그먼트 플러그인 경로, dify_plugin DB는 짧은 이름. mart의 형식은 collector가 dify DB에서 읽은 값을 그대로 저장하므로 플러그인 경로일 가능성 높으나, 미확인.

### 4.8 조사 4에서 확인된 사실 (팩트만)

> 아래는 **조사 시점(2026-06-10) 운영 DB 스냅샷**이며 provider 구성은 수시 변동됨. 특정 비중·호출량 기반의 판단은 이 문서 범위 밖.

1. vLLM은 독립 플러그인 provider `yangyaofei/vllm/vllm`으로 등록되어 있음 (openai_api_compatible이 아님).
2. `openai_api_compatible`은 embeddings(bge-m3-ko) 1건만 등록.
3. **provider 이름이 플러그인 경로 형식** (`langgenius/ollama/ollama`) — `LOCAL_PROVIDERS`의 짧은 이름(`ollama`)과 **형식 불일치** 가능성.
4. `xinference`, `localai`, `openllm`, `text-generation-inference` → 이 DB에서는 미등록.
5. `providers` 테이블 구조: `encrypted_config` 컬럼 제거 → `credential_id` FK로 교체됨.

---

## 5. 조사 5 — audit/대시보드 경로

### 5.1 LOCAL_PROVIDERS 정의 위치 (코드 근거)

| 파일 | 라인 | 변수명 | 사용 위치 |
|------|------|--------|-----------|
| `api/services/admin/dashboard_model_tokens_service.py` | **:19** | `LOCAL_PROVIDERS` | `:58` `get_model_tokens()` |
| `api/services/admin/dashboard_drill_calls_service.py` | **:49** | `_LOCAL_PROVIDERS` | `:82` `get_model_call_share()` |

> **중복 정의** — 2곳에 동일 frozenset이 별도 선언됨. 공유 상수화 권장.

### 5.2 현재 데이터 흐름

```
workflow_node_executions (Dify DB)
  → dify-audit collector (provider명 + model명만 수집)
    → audit_events / mart 테이블 (provider + model — endpoint 없음)
      → dashboard service (LOCAL_PROVIDERS 이름 매칭, 조회 시점)
        → 프론트엔드 "(로컬)" 라벨
```

**collector 수집 내용** (`dify-audit/src/lib/collectors/workflow-nodes.ts`):
- `provider` (이름 문자열), `model` (모델명), token 수치
- endpoint/base_url/config **미수집**

**provider-changes collector** (`dify-audit/src/lib/collectors/provider-changes.ts`):
- `provider_name`, `model_name`, `model_type`, `is_valid` — 설정 변경 이벤트만
- endpoint/config **미수집**

**mart 테이블** (`spx_mv_model_tokens_daily`, `api/models/mart.py:115-134`):
- `model_provider` (Text), `model_id` (Text), `tokens`, `calls` — endpoint 없음

### 5.3 접근법 비교

| 접근법 | 위치 | 실현성 | 장점 | 단점 |
|--------|------|--------|------|------|
| **(A) Collector 수집 시 `isLocal` 주입** | dify-audit collector | **낮음** | 1회 분류, 조회 비용 없음 | RSA 암호화 키 접근 필요 (TS에서 PY 암호화 복호화), 설정 변경 시 과거 데이터 불일치 |
| **(B) Dashboard 서비스 JOIN** | api/services/admin/ | **중간** | 항상 최신 반영, Flask에서 Dify 복호화 유틸 사용 가능 | 매 조회마다 복호화 비용, 캐시 필요 |
| **(C) LOCAL_PROVIDERS 확장** | api/services/admin/ | **높음** | 즉시 적용, 코드 변경 최소 | `openai_api_compatible` 커버 불가 |
| **(D) 설정 기반 allowlist** | 환경변수 또는 DB 설정 | **높음** | `openai_api_compatible` 해결 가능, 관리 유연 | 관리자 수동 설정 필요, 설정 UI/관리 오버헤드 |

---

## 6. 신호 비교표 (종합 — DB 실증 반영)

| 신호 | 커버리지 | 정확도 | 데이터 가용성 | 구현 비용 | 비고 |
|------|----------|--------|-------------|----------|------|
| **이름 목록 (현재 LOCAL_PROVIDERS)** | **최저** — 짧은이름 `ollama`이지만 Dify DB에는 `langgenius/ollama/ollama`로 저장 → mart도 같은 형식이면 **형식 불일치** | - | - | - | mart 형식 미확인 (§ 9.3) |
| **플러그인 경로 부분 매칭** | 높음 — `contains("ollama")` 또는 경로 마지막 세그먼트 추출 | 높음 | 즉시 | 낮음 | `yangyaofei/vllm/vllm` → `vllm` 추출 |
| **플러그인 경로 allowlist** | 높음 — 전체 경로로 정확 매칭 | **최고** | 즉시 | 낮음 | 플러그인 변경 시 갱신 필요 |
| **configurate_methods=customizable-model** | 넓음 | **중간** (openai_api_compatible도 customizable) | `dify_plugin` DB JSON 파싱 | 중간 | 보조 신호로 활용 가능 (§ 2.2) |
| **provider_type=custom** | 넓음 | **낮음** (자격증명 범위이며 로컬/클라우드 구분 아님 — § 2.1 참조) | 즉시 | 최저 | 사용 불가 |
| **endpoint URL (private IP)** | 높음 | **최고** | 암호화 → 복호화 필요 | 높음 | - |

---

## 7. 권고안

### 확인된 기술적 사실 기반 선택지

조사에서 확인된 사실:
- provider 이름이 플러그인 경로 형식(`langgenius/ollama/ollama`)으로 저장됨
- 현재 `LOCAL_PROVIDERS`는 짧은 이름(`ollama`) → 형식 불일치 가능성
- vLLM은 독립 플러그인으로 존재 가능 (DB에서 확인)
- `openai_api_compatible`로 LLM을 등록하는 경우도 가능 (이름으로 판정 불가)
- endpoint URL은 암호화되어 SQL 질의 불가

이를 바탕으로 가능한 구현 방식:

#### 방법 A: 플러그인 경로 마지막 세그먼트 추출 + 이름 매칭

```python
LOCAL_PROVIDERS = frozenset({"ollama", "xinference", "localai", "vllm", "openllm"})

def _is_local_provider(provider_path: str) -> bool:
    """플러그인 경로 형식(author/plugin/provider)에서 마지막 세그먼트 추출 후 매칭."""
    short_name = provider_path.rsplit("/", 1)[-1] if "/" in provider_path else provider_path
    return short_name in LOCAL_PROVIDERS
```

- 장점: 간결, 플러그인 author/경로 변경에 강건, 기존 frozenset 확장만으로 작동
- 단점: `openai_api_compatible`로 등록된 로컬 LLM은 커버 불가

#### 방법 B: 전체 경로 allowlist

```python
LOCAL_PROVIDER_PATHS = frozenset({
    "langgenius/ollama/ollama",
    "yangyaofei/vllm/vllm",
})

def _is_local_provider(provider_path: str) -> bool:
    return provider_path in LOCAL_PROVIDER_PATHS
```

- 장점: 정확한 매칭
- 단점: 플러그인 변경·추가 시 갱신 필요

#### 적용 대상 (2곳 — 공유 상수화 권장)

1. `api/services/admin/dashboard_model_tokens_service.py:19` — `LOCAL_PROVIDERS`
2. `api/services/admin/dashboard_drill_calls_service.py:49` — `_LOCAL_PROVIDERS`

---

## 8. openai_api_compatible 처리 방침

### 현재 상태

- `openai_api_compatible`은 로컬 vLLM도, 원격 서비스도 같은 provider명으로 등록 가능 — 이름만으로 판정 불가.
- 운영 DB 스냅샷에서는 embeddings만 등록되어 있었으나, 이는 특정 시점의 구성이므로 일반화할 수 없음.
- vLLM이 독립 플러그인으로 등록될 수도, `openai_api_compatible`로 등록될 수도 있음 — **환경에 따라 다름**.

### 선택지

| 방침 | 설명 | 위험 |
|------|------|------|
| **(A) 미표시 감수** | `openai_api_compatible`은 "(로컬)" 라벨 안 붙임 | 로컬 모델 누락 가능 |
| **(B) 전부 로컬로 간주** | 환경변수로 제어 | 원격 서비스 등록 시 오분류 |
| **(C) endpoint URL 복호화** | 런타임에 base_url 확인 | 구현 비용 높음 |
| **(D) 설정 allowlist** | 관리자가 지정 | 수동 관리 필요 |

---

## 9. PM 결정 필요 항목

1. **`LOCAL_PROVIDERS`에 추가할 provider 범위**
   - vllm 추가 여부 (정의상 로컬 자체호스팅)
   - 그 외 후보: openllm, text-generation-inference 등

2. **플러그인 경로 매칭 방식 결정**
   - 방법 A (마지막 세그먼트 추출) vs 방법 B (전체 경로 allowlist)

3. **mart의 `model_provider` 형식 확인**
   - 플러그인 경로(`langgenius/ollama/ollama`)인지 짧은 이름(`ollama`)인지 실제 mart 데이터로 확인 필요
   - 불일치 시 기존 "(로컬)" 라벨 동작 여부 점검 대상

4. **`openai_api_compatible` 처리 방침**
   - 환경에 따라 로컬/원격 혼재 가능 — 미표시 감수 vs 별도 설정
