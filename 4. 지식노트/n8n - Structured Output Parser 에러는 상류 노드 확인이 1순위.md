---
tags: [지식, n8n, LLM, 디버깅, 진단방법론, AI-Agent]
date: 2026-05-27
---
# n8n - Structured Output Parser 에러는 상류 노드 확인이 1순위

## 핵심

- n8n의 `"Model output doesn't fit required format"` 에러는 **파서 자체 문제가 아니라 상류 노드(LLM 모델)의 출력 형식 문제**인 경우 다수
- **에러 발생 노드보다 상류를 먼저 확인** — 파서는 받은 텍스트를 파싱 시도할 뿐, 텍스트 자체가 깨졌으면 파서 설정을 아무리 바꿔도 안 풀림
- 같은 패턴이 LangChain `OutputParser`, Dify Agent의 tool call 인식 실패에도 적용됨

---

## 1. 안티패턴 — 파서만 들여다보는 디버깅

Structured Output Parser 에러를 보면 흔히 시도하는 것:

- ❌ 파서의 JSON schema 수정
- ❌ 파서 옵션 변경 (strict / lenient)
- ❌ 프롬프트에 "JSON으로 응답하라" 강조
- ❌ 파서 노드 자체 교체 (다른 OutputParser로)

이 시도들은 **상류 노드가 정상 텍스트를 주고 있다는 전제** 하에서만 유효. 전제가 깨져있으면 무의미.

---

## 2. 올바른 진단 순서

### 2-1. 상류 노드(LLM) 실제 출력 확인

n8n 실행 결과 보기:

```
워크플로우 실행 → 상류 OpenAI Chat Model 노드 클릭 → Output 탭
```

기대 형식 (정상):

```json
{
  "message": {
    "content": null,
    "tool_calls": [
      {"function": {"name": "...", "arguments": "{...}"}}
    ]
  }
}
```

비정상 패턴 예시 (본 케이스):

```
call:Price_Tool_research{URL:https://query1.finance.yahoo.com/v8/finance/chart/GC=F?range=1d&interval=1d}
```

→ 텍스트로 노출. 파서가 받기 전에 이미 깨져있음.

### 2-2. 상류 노드의 모델·API 경로 확인

- 어떤 모델인가? (`gemma-4-26b` 같은 비-OpenAI 모델은 tool call 출력 포맷이 비표준)
- 어떤 API 경로인가? (`use_responses_api` 옵션 — Responses API vs Chat Completions)
- 같은 워크플로우의 다른 OpenAI 호환 노드는 어떻게 출력하는가?

### 2-3. 모델 서버 직접 호출 (curl)

n8n 거치지 않고 모델 서버에 동일 요청:

```bash
curl http://<vllm-server>:8003/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{...같은 요청...}'
```

- 정상 `tool_calls` 반환 ⇒ n8n 설정 문제 (예: `use_responses_api` 옵션)
- 동일하게 raw text 반환 ⇒ 모델/추론 엔진 자체 문제 (parser 미설정 등)

---

## 3. 본 케이스 — 5/27 시장 조사 워크플로우

| 단계 | 결론 |
|------|------|
| 1차 의심 | Structured Output Parser schema → ❌ 무관 |
| 2차 의심 | LLM 모델 응답 텍스트 자체 깨짐 → ✅ 맞음 |
| 3차 의심 | vLLM gemma4 parser 미동작 → 모델 출력 raw 노출 → ✅ 확정 |
| 해결 | n8n `use_responses_api: false` → Chat Completions 경로 → vLLM gemma4 parser 정상 개입 |

상세: [[vLLM - Responses API tool calling 버그 (gemma4 parser 미개입)]]

---

## 4. 일반화

LLM 기반 워크플로우 디버깅의 일반 원칙:

```
파서 에러 발생 → 파서가 받은 입력 확인 → 그 입력이 어디서 왔는지 추적
```

**상류 → 하류 순서로 디버깅하지 말 것**. 에러 노드에서 거꾸로 상류 추적이 정공법.

비슷한 패턴이 적용되는 곳:

| 도구 | 에러 패턴 | 상류 확인할 것 |
|------|-----------|----------------|
| n8n | `Model output doesn't fit required format` | LLM 노드 출력 텍스트 |
| LangChain | `OutputParserException` | LLM 호출 raw 응답 |
| Dify Agent | `tool_responses: []` (tool 인식 실패) | Dify가 vLLM에 보낸 요청 + 받은 응답 |
| OpenAI Function Calling | `JSONDecodeError` | `arguments` 필드 raw |

---

## 5. 학습 포인트

- **에러 발생 위치 ≠ 에러 원인 위치** — 파이프라인에서는 상류가 깨졌어도 에러는 하류에서 터짐
- LLM 워크플로우는 자연어/JSON 혼재 흐름이라 **상류 출력의 raw 텍스트를 항상 우선 확인**
- 파서를 바꿔서 해결되는 케이스는 드물다 — 대부분 모델 설정 / API 경로 / 프롬프트가 원인
- 본 케이스에서 1차 가설(파서 schema)부터 들어갔다면 vLLM 소스 추적까지 못 갔을 것

---

## 관련 노트

- [[vLLM - Responses API tool calling 버그 (gemma4 parser 미개입)]]
- [[Dify Agent - gemma-4 vLLM Tool Calling 트러블슈팅]]
- [[LLM - JSON Over-escape 버그와 복구 패턴]]
- [[Dify Agent - LLM 카테고리명 환각 대응]]
