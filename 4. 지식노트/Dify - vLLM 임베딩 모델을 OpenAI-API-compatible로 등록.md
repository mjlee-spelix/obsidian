---
tags: [지식, dify, vLLM, 임베딩, RAG, AI, 트러블슈팅]
date: 2026-05-27
---
# Dify - vLLM 임베딩 모델을 OpenAI-API-compatible로 등록

## 핵심

- **vLLM 플러그인(`yangyaofei/vllm`)은 전 버전(0.1.3~0.2.3)에서 TEXT EMBEDDING 타입 미지원** — `manifest.yaml`에 `text_embedding: false`로 박혀있음
- **OpenAI 제공자 직접 등록도 실패** — Validate Model 단계에서 임베딩 전용 서버라 Chat Completions 검증 실패
- ✅ **표준 우회 패턴: "OpenAI-API-compatible" 플러그인 설치 후 TEXT EMBEDDING 타입으로 개별 등록**
- vLLM은 OpenAI 호환 `/v1/embeddings` 엔드포인트를 제공하므로 OpenAI-API-compatible 제공자가 그대로 사용 가능

---

## 1. 배경

'사규 검색' 에이전트 RAG 임베딩 모델 교체:

- 기존: `qwen3-embedding:8b` (Ollama) — **서버에서 모델 소실** (Dify UI에는 보이지만 실제 Ollama 서버에 없는 상태)
- 신규: `bge-m3-ko` (vLLM 서빙, `http://192.168.10.40:8010`)

---

## 2. 시도 ① vLLM 플러그인 사용 (실패)

Dify의 `yangyaofei/vllm` 플러그인을 통해 임베딩 모델 등록을 시도했으나:

- 플러그인이 TEXT EMBEDDING **모델 타입을 노출하지 않음**
- 0.1.3, 0.2.0, 0.2.3 전 버전 확인 → 모두 동일

확정 검증:

```bash
# 플러그인 manifest 확인
cat plugins/yangyaofei__vllm/manifest.yaml | grep text_embedding
# text_embedding: false
```

→ 플러그인이 LLM/Reasoning 타입만 제공, embedding은 미지원

---

## 3. 시도 ② OpenAI 제공자 직접 등록 (실패)

`Models → Add Model Provider → OpenAI` 사용:

- API Base를 vLLM 임베딩 서버로 지정 (`http://192.168.10.40:8010`)
- **"Validate Model" 단계에서 실패** — OpenAI 제공자는 Chat Completions API로 헬스체크하는데, vLLM 임베딩 서버는 `/v1/embeddings`만 서빙

→ OpenAI 제공자는 **임베딩 전용 서버에 못 붙음**

---

## 4. ✅ 표준 우회 패턴 — OpenAI-API-compatible 플러그인

### 4-1. 플러그인 설치

`Plugins → Marketplace → "OpenAI-API-compatible"` 검색 후 설치

### 4-2. TEXT EMBEDDING 타입으로 개별 등록

`Models → Add Model Provider → OpenAI-API-compatible → Add Model`

| 항목 | 값 | 비고 |
|------|-----|------|
| Model Type | **TEXT EMBEDDING** | 핵심 — 별도 선택 가능 |
| Model Name | `bge-m3-ko` | vLLM `--served-model-name`과 일치 |
| API Key | (vLLM 미보호면 임의값 OK) | |
| API Base | `http://192.168.10.40:8010` | **`/v1` 제외 — Dify가 자동 추가** |
| Context Size | `8192` | vLLM `--max-model-len`과 일치 |
| Max chunks per batch | `256` | |

⚠️ **API Base에 `/v1` 넣지 말 것** — Dify가 자동 추가하므로 이중 추가됨 (`/v1/v1/...`)

### 4-3. 검증

```
Knowledge → 데이터셋 설정 → 임베딩 모델 = bge-m3-ko 선택
→ "검색 테스트" 실행 → 응답 정상 받으면 OK
```

---

## 5. 차원/모델명 mismatch 주의

기존 인덱스가 옛 임베딩 모델로 생성된 벡터라면 **재인덱싱 필요**:

- 본 케이스에서 임베딩 교체 후 검색 스코어 `0.11~0.13`으로 급락 — 차원/모델 mismatch 영향
- 일반적으로는 **재인덱싱이 정공법**, 단 본 케이스는 담당자 트랙 선택으로 미수행
- 운영 환경에서 임베딩 모델을 바꾸려면 **인덱싱·검색 양쪽 정합**이 필수

→ [[임베딩 모델 교체 시 인덱싱·검색 양쪽 정합 필수]] (작성 예정)

---

## 6. vLLM 임베딩 서버 띄울 때

```bash
vllm serve <model-path> \
  --served-model-name bge-m3-ko \
  --port 8010 \
  --max-model-len 8192 \
  --task embedding   # 경우에 따라 필요
```

- `--task embedding` 플래그가 필요할 수 있음 (이번 케이스는 없이도 동작)
- vLLM은 OpenAI 호환 `/v1/embeddings` 엔드포인트를 자동 노출

---

## 7. 학습 포인트

- Dify 플러그인이 특정 모델 타입을 미지원해도, **OpenAI 호환 API를 제공하는 서버라면 OpenAI-API-compatible 플러그인으로 우회 가능**
- 임베딩 전용 서버에는 일반 OpenAI 제공자가 못 붙음 — Chat Completions 검증 실패
- Dify의 API Base에 `/v1` 넣으면 이중 추가 (`/v1/v1/...`) — **항상 제외**
- 플러그인 manifest의 모델 타입 지원 여부는 `text_embedding`, `llm`, `reasoning` 등 boolean으로 박혀있음

---

## 관련 노트

- [[Dify Agent - gemma-4 vLLM Tool Calling 트러블슈팅]]
- [[vLLM - Responses API tool calling 버그 (gemma4 parser 미개입)]]
- [[Dify - 데이터 소스 3종 비교 (OLTP vs 로그파일 vs 내부 로그 테이블)]]
