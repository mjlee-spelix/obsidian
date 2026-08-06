---
tags: [지식, dify, AI-Agent, LLM, vLLM, gemma, function-calling, 트러블슈팅]
date: 2026-04-23
---
# Dify Agent - gemma-4 vLLM Tool Calling 트러블슈팅

## 핵심
- Dify Agent에서 gemma-4-26b(vLLM) 사용 시 tool call이 raw 토큰으로 노출되는 문제
- vLLM curl 직접 테스트에서는 `tool_calls` 정상 파싱 확인 → **문제는 Dify ↔ vLLM 사이**
- 미해결 상태 — Dify 설정/연동 원인 추가 조사 필요

---

## 1. 환경

| 항목 | 값 |
|------|-----|
| 모델 | gemma-4-26B-A4B-it |
| 추론 엔진 | vLLM (`--tool-call-parser gemma4 --reasoning-parser gemma4`) |
| 서버 | 192.168.10.40:8003 |
| 플랫폼 | Dify Agent (function calling 전략) |
| served-model-name | `/install_file_backup/tessinu/gemma-4-26b` |

---

## 2. 증상

Dify Agent에서 gemma-4로 tool calling 시:

```json
{
  "output": {
    "llm_response": "<|tool_call>call:SubAgentCompanyRegulationsRAG{query:<|\"|>연차 규",
    "tool_responses": []
  }
}
```

- `<|tool_call>`, `<|"|>` 등 gemma의 raw 특수 토큰이 텍스트로 노출
- `tool_responses`가 빈 배열 → Dify가 tool call을 인식하지 못함
- 응답이 중간에 잘림 ("연차 규"에서 끊김)

---

## 3. 진단 과정

### 3-1. 서버 프로세스 확인 (`ps aux | grep 8003`)

포트 8003에 **프로세스 4개가 중복 실행** 중이었음:

| PID | 모델 | TP | 시작일 | 상태 |
|-----|------|----|--------|------|
| 2451353 | Qwen3.5-27B | 1 | Apr17 | 좀비 (추정) |
| 3728148 | Qwen3.5-27B | 2 | Apr17 | 좀비 (추정) |
| 2629016 | gemma-4-26B | 2 | Apr18 | 좀비 (추정) |
| 2836355 | gemma-4-26B | 1 | Apr20 | **활성** |

→ 좀비 프로세스 정리 필요 (`kill 2451353 3728148 2629016`)

parser 설정 확인 결과, 활성 프로세스에 `--tool-call-parser gemma4 --reasoning-parser gemma4`가 이미 적용되어 있었음.

### 3-2. vLLM 직접 테스트 (`curl`)

Dify를 거치지 않고 vLLM에 직접 tool calling 요청:

```bash
curl http://localhost:8003/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/install_file_backup/tessinu/gemma-4-26b",
    "messages": [
      {"role": "user", "content": "연차 규정 알려줘"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "SubAgentCompanyRegulationsRAG",
          "description": "회사 규정을 조회합니다",
          "parameters": {
            "type": "object",
            "properties": {
              "query": {"type": "string", "description": "검색할 규정 내용"}
            },
            "required": ["query"]
          }
        }
      }
    ]
  }' 2>/dev/null | python3 -m json.tool
```

**결과:**

```json
{
    "choices": [
        {
            "message": {
                "content": null,
                "tool_calls": [
                    {
                        "id": "chatcmpl-tool-a321c33dbef56d46",
                        "type": "function",
                        "function": {
                            "name": "SubAgentCompanyRegulationsRAG",
                            "arguments": "{\"query\": \"연차 규정\"}"
                        }
                    }
                ]
            },
            "finish_reason": "tool_calls"
        }
    ]
}
```

| 필드 | 값 | 상태 |
|------|-----|------|
| `content` | `null` | ✅ 토큰 안 새고 있음 |
| `tool_calls` | 정상 파싱 | ✅ 이름, 파라미터 정확 |
| `finish_reason` | `"tool_calls"` | ✅ tool call로 정상 종료 |
| `arguments.query` | `"연차 규정"` | ✅ 잘리지 않음 |

→ **vLLM의 `gemma4` parser가 완벽하게 동작**. 문제는 Dify 쪽으로 특정됨.

### 3-3. Dify 모델 설정 확인

Dify의 "Function Call Type" 설정:

| 옵션 | API 요청 | API 응답 읽는 필드 |
|------|----------|-------------------|
| **Function Call** (구 방식) | `functions`, `function_call` | `message.function_call` |
| **Tool Call** (신 방식) | `tools`, `tool_choice` | `message.tool_calls` |
| Not Support | - | - |

vLLM은 `tool_calls` 배열로 응답하므로 "Tool Call"이 맞음.
"Function Call" → "Tool Call"로 변경 시도했으나 **여전히 안 됨**.

---

## 4. 현재 결론

```
vLLM gemma4 parser → ✅ 정상
Dify ↔ vLLM 연동 → ❌ 문제 지점
```

Dify가 vLLM에 `tools` 파라미터를 제대로 안 보내거나, 응답의 `tool_calls` 필드를 못 읽는 것으로 추정.

---

## 5. 미해결 — 추가 확인 필요

- [ ] Dify가 실제로 보내는 API 요청 확인 (vLLM 로그에서 `tools` 포함 여부)
  ```bash
  tail -200 vllm_8003.log
  ```
- [ ] Dify api 컨테이너 로그 확인
  ```bash
  docker compose logs api --tail 100 | grep -i "tool\|error\|gemma"
  ```
- [ ] 모델 삭제 후 재등록 (설정 캐시 초기화)
- [ ] 좀비 프로세스 정리 후 재테스트
- [ ] vLLM 환경 이슈 확인 (zmq `GLIBCXX_3.4.30` 에러 — 서빙에는 영향 없지만 불안정 신호)

---

## gpt-oss 트러블슈팅과 비교

| | gpt-oss | gemma-4 |
|--|---------|---------|
| 문제 토큰 | `<\|call\|>`, `<\|channel\|>` | `<\|tool_call>`, `<\|"\|>` |
| vLLM parser | `openai` + `reasoning-parser openai_gptoss` | `gemma4` + `reasoning-parser gemma4` |
| vLLM 레벨 파싱 | ✅ 정상 | ✅ 정상 |
| Dify 연동 | ✅ 동작 (코드 노드 방어 필요) | ❌ tool call 자체를 인식 못함 |
| 핵심 차이 | Dify가 tool_calls는 읽지만 content에 가짜 답변이 섞임 | Dify가 tool_calls 자체를 못 읽음 |

→ gpt-oss보다 앞 단계에서 막혀 있음. Dify 모델 provider 설정이 근본 원인일 가능성.

---

## 관련 노트
- [[Dify Agent - gpt-oss vLLM Function Calling 트러블슈팅]]
- [[gpt-oss vLLM Function Calling - 원인 분석 심층 리포트]]
- [[목적별 모델 비교 테스트 설계]]
