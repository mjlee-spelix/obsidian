---
tags: [지식, vLLM, LLM, gemma, function-calling, 트러블슈팅, n8n]
date: 2026-05-27
---
# vLLM - Responses API tool calling 버그 (gemma4 parser 미개입)

## 핵심

- 같은 vLLM 서버에서 **`/v1/chat/completions` 경로는 tool calling 정상, `/v1/responses` 경로는 raw text 노출**
- 비스트리밍·스트리밍 양쪽 모두 실패 — gemma4 전용 tool call parser가 Responses API 경로에서는 개입하지 않음
- **운영 지침: vLLM + 비-OpenAI 모델 조합에서 tool calling이 깨지면 `use_responses_api: false`로 Chat Completions 경로 전환**
- 같은 vLLM 서버라도 **모델 교체 시 API 경로별 동작 검증이 필수** (gpt-oss-120b는 양쪽 OK, gemma-4-26b는 한쪽만 OK)

---

## 1. 환경

| 항목 | 값 |
|------|-----|
| 모델 | `gemma-4-26b` (비표준 자체 tool 포맷) |
| 추론 엔진 | vLLM (`--enable-auto-tool-choice --tool-call-parser gemma4 --reasoning-parser gemma4`) |
| 서버 | `http://192.168.10.40:8003` (conda `py312_sr`) |
| 클라이언트 | n8n `2.6.3` Self-Hosted, OpenAI Chat Model 노드 |
| 워크플로우 | `Autonomous Market Research Agent` (시장 조사 + 뉴스레터) |

---

## 2. 증상

- n8n Structured Output Parser에서 `"Model output doesn't fit required format"` 에러 반복 (워크플로우 2사이클, 총 6회)
- OpenAI Chat Model 노드가 tool call을 구조화된 `tool_calls` 필드가 아닌 일반 텍스트로 반환:

```
call:Price_Tool_research{URL:https://query1.finance.yahoo.com/v8/finance/chart/GC=F?range=1d&interval=1d}
```

- 일부 입력은 `[object Object]`로 전달됨 (JS 객체 → 문자열 변환 부산물)

---

## 3. 진단 흐름

### 3-1. 1차 가설 (폐기) — chat template tool 구조 불일치

- Responses API: tool 정의 평탄 구조 (`tool_data['name']`)
- Chat Completions: 중첩 구조 (`tool_data['function']['name']`)
- gemma chat template이 중첩만 처리해서 Responses API에서 tool 정의가 누락된다는 가설
- **폐기**: gpt-oss-120b의 chat template도 동일하게 `tool.function` 중첩 구조를 기대 → template 차이가 아님

### 3-2. 2차 가설 — vLLM 소스 코드 레벨 추적

- vLLM 설치 경로: `/root/miniconda3/envs/py312_sr/lib/python3.12/site-packages/vllm/`
- `ParserManager.get_parser()` (Responses 경로) vs `ParserManager.get_tool_parser()` (Chat Completions 경로) 초기화 차이 확인
- `_WrappedParser.__init__`에서 `tool_parser_cls(tokenizer)` — **tools 미전달 발견** (비스트리밍 경로)
- 스트리밍 경로에서는 `tool_parser_cls(tokenizer, request.tools)`로 tools 전달 코드 존재

### 3-3. 3차 검증 — curl로 경로별 직접 테스트

동일 요청을 3가지 경로로 전송:

| 경로 | 결과 |
|------|------|
| `/v1/chat/completions` 비스트리밍 | `tool_calls` 필드에 구조화된 tool call 정상 반환 ✅ |
| `/v1/responses` 비스트리밍 | `"text": "call:get_price{ticker:GC=F}"` — 텍스트로 반환 ❌ |
| `/v1/responses` 스트리밍 | `response.output_text.delta`로 텍스트 반환 — `function_call` 이벤트 아님 ❌ |

---

## 4. 확정된 원인

**vLLM의 Responses API 경로(`/v1/responses`)에서 gemma4 tool call parser가 정상 동작하지 않는 버그.**

- 모델은 양쪽 경로 모두 동일한 출력 생성 (`<|tool_call>call:get_price{ticker:GC=F}<tool_call|>`)
- Chat Completions 경로: gemma4 전용 tool call parser가 이 출력을 파싱 → 구조화된 `tool_calls` 반환
- Responses API 경로: tool call parser가 개입 못함 → raw text 그대로 반환
- 비스트리밍·스트리밍 모두 동일하게 실패

---

## 5. gpt-oss-120b는 왜 Responses API에서도 동작했나

- gpt-oss-120b는 tool call 출력이 **JSON 기반**이고 OpenAI 형식에 가까움
- vLLM 파서가 어느 경로든 처리 가능
- gemma-4-26b는 **비표준 자체 포맷** (`call:Name{key:value}`) — 전용 `gemma4` 파서 필수인데, Responses API 경로에서는 이 파서가 동작하지 않음

→ **모델 출력 포맷이 OpenAI 표준에 가까울수록 API 경로 호환성이 높다**는 일반화 가능

---

## 6. 해결

n8n OpenAI Chat Model 노드에서 **`use_responses_api: false` 설정** → Chat Completions API 경로로 전환

```
OpenAI Chat Model 노드 (use_responses_api: false)
JSON Model 노드      (use_responses_api: false)
```

- ⚠️ **노드별 개별 설정 필요** — n8n의 `use_responses_api`는 글로벌 옵션이 아니라 각 모델 노드 인스턴스의 옵션
- 같은 워크플로우에 OpenAI 호환 노드가 여러 개면 **모두** 개별 변경

---

## 7. Responses API vs Chat Completions API

| | Responses API | Chat Completions API |
|---|---|---|
| 경로 | `/v1/responses` | `/v1/chat/completions` |
| 출시 | OpenAI 2025년 3월 | 2023~ |
| n8n 옵션 | `use_responses_api: true` | `use_responses_api: false` |
| vLLM 지원 | v0.11.0+ (신규) | 성숙 |
| 알려진 vLLM 버그 | JSON schema 누출 #38245, truncation #38132 등 다수 | 모델별 전용 parser 안정 |

**운영 지침**: vLLM + 비-OpenAI 모델 조합에서 tool calling 문제 발생 시 **첫 번째 시도 = `use_responses_api: false`**

---

## 8. 학습 포인트

- 같은 vLLM 서버에서 **모델만 바꿔도 API 경로별 동작이 달라질 수 있음** — 모델 교체 시 API 호환성도 함께 검증 필요
- vLLM의 Chat Completions 경로는 모델별 전용 parser가 안정적으로 동작하지만, Responses API 경로는 상대적으로 새 코드라 모델별 호환성 이슈 존재
- Structured Output Parser 에러는 파서 자체 문제가 아니라 **상류 노드(모델)의 출력 형식 문제**인 경우가 많음 → [[n8n - Structured Output Parser 에러는 상류 노드 확인이 1순위]]
- vLLM 소스 코드 직접 추적이 필요할 때 conda 환경 경로 확인: `/root/miniconda3/envs/<env>/lib/python<ver>/site-packages/vllm/`

---

## 관련 노트

- [[Dify Agent - gemma-4 vLLM Tool Calling 트러블슈팅]]
- [[Dify Agent - gpt-oss vLLM Function Calling 트러블슈팅]]
- [[gpt-oss vLLM Function Calling - 원인 분석 심층 리포트]]
- [[n8n - Structured Output Parser 에러는 상류 노드 확인이 1순위]]
