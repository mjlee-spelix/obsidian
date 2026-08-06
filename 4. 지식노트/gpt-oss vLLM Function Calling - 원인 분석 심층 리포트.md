# gpt-oss vLLM Function Calling - 원인 분석 심층 리포트

> 작성일: 2026-03-31
> 관련 문서: [[Dify Agent - gpt-oss vLLM Function Calling 트러블슈팅]]

---

## 1. 개요

Dify Agent + gpt-oss-120b + vLLM 환경에서 발생하는 두 가지 문제의 **정확한 원인**을 vLLM API 직접 호출(curl) 및 소스코드 분석을 통해 규명한 리포트.

| 문제 | 현상 | 원인 레이어 |
|------|------|------------|
| JSON 중복 출력 | 가짜 답변 JSON + 실제 답변 JSON이 합쳐져 나옴 | 모델 + Dify 호환성 |
| 병렬 tool call | 동일 쿼리가 1~6회 중복 호출 | 모델 |

---

## 2. vLLM API 직접 테스트 (curl)

### 목적

Dify를 거치지 않고 vLLM에 직접 요청하여, 문제가 vLLM/모델에서 발생하는지 Dify에서 발생하는지 분리.

### 방법

```bash
# vLLM 서버(192.168.10.40)에 SSH 접속 후 실행
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/install_file_backup/tessinu/gpt-oss-120b",
    "messages": [
      {"role": "user", "content": "올해 월별 매출을 알려줘"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "sql_executor_tool",
          "description": "SQL 쿼리를 실행합니다",
          "parameters": {
            "type": "object",
            "properties": {
              "query": {"type": "string", "description": "실행할 SQL 쿼리"}
            },
            "required": ["query"]
          }
        }
      }
    ]
  }' 2>/dev/null | python3 -m json.tool
```

> tools는 실제 실행이 아니라 모델에게 "이런 도구가 있다"는 스키마를 알려주는 것. 모델은 호출 의도(intent)만 반환하고, 실제 실행은 클라이언트(Dify)가 담당.

### 결과

```json
{
  "choices": [{
    "message": {
      "content": "아래는 2026년(올해) 각 월별 매출 총액입니다...(가짜 데이터)",
      "tool_calls": [
        {"function": {"name": "sql_executor_tool", "arguments": "{\"query\": \"SELECT ...\"}"}},
        {"function": {"name": "sql_executor_tool", "arguments": "{\"query\": \"SELECT ...\"}"}}
      ],
      "reasoning_content": "The user asks...We need to query...Let's imagine: 120000...",
      "reasoning": "(reasoning_content와 동일)"
    },
    "finish_reason": "tool_calls"
  }]
}
```

### 핵심 발견

| 필드 | 내용 | 상태 |
|------|------|------|
| `reasoning_content` | 영문 CoT (추론 과정) | 정상 분리됨 (reasoning_parser 작동) |
| `tool_calls` | SQL 쿼리 2개 (병렬) | 파싱 정상, 개수가 문제 |
| `content` | 한글 가짜 답변 (상상 데이터) | **이것이 문제의 원인** |

`tool_choice: "required"` 옵션을 추가해도 동일한 결과 — content에 가짜 답변이 여전히 생성됨.

---

## 3. gpt-oss 채널 구조

### 3채널 시스템

gpt-oss는 `chat_template.jinja`에 정의된 3개의 출력 채널을 사용:

```
# Valid channels: analysis, commentary, final.
# Channel must be included for every message.
# Calls to these tools must go to the commentary channel: 'functions'.
```

| 채널 | 용도 | vLLM API 매핑 |
|------|------|--------------|
| `analysis` | CoT/추론 과정 | `reasoning_content` |
| `commentary` | 도구 호출 (functions) | `tool_calls` |
| `final` | 사용자에게 보여줄 최종 답변 | `content` |

### 모델 생성 시 raw 출력 구조

```
<|start|>assistant<|channel|>analysis<|message|>
We need to query the database...
<|end|>

<|start|>assistant to=functions.sql_executor_tool<|channel|>commentary json<|message|>
{"query": "SELECT ..."}
<|call|>
                            ← 여기서 멈춰야 함

<|start|>assistant<|channel|>final<|message|>
아래는 2026년 월별 매출입니다... 120,000...
<|end|>
                            ← 그런데 여기까지 생성해버림
```

---

## 4. 문제 1: content에 가짜 답변이 들어가는 원인

### 모델 측 (근본 원인)

모델이 tool call을 할 때 `analysis → commentary`에서 멈춰야 하는데, **final 채널까지 생성**함.

`reasoning_content`를 보면 원인이 명확:

> "Assume the function returns something like rows. **Let's imagine:** 2026-01 | 120000..."

모델이 도구 결과를 기다리지 않고 **상상해서 답변을 완성**하는 행동. 이는 모델의 학습/fine-tuning에서 비롯된 특성이며, API 파라미터나 서버 설정으로 제어 불가.

**GPT-4와의 차이:**

| | GPT-4 | gpt-oss |
|--|-------|---------|
| tool_call 턴의 content | `null` | 가짜 답변 (상상 데이터) |
| 영향 | 없음 | Dify 누적 시 문제 발생 |

### Dify 측 (확대 원인)

`api/core/agent/fc_agent_runner.py`의 에이전트 루프:

```python
final_answer = ""   # 루프 시작 전 초기화

while function_call_state and iteration_step <= max_iteration_steps:
    response = ""

    # content 수집 — tool_calls 여부와 상관없이 무조건
    if chunk.delta.message and chunk.delta.message.content:
        response += str(chunk.delta.message.content)

    # tool_calls 수집
    if self.check_tool_calls(chunk):
        function_call_state = True

    # 핵심: 조건 분기 없이 무조건 누적
    final_answer += response + "\n"    # ← tool_calls 턴의 content도 포함됨

    iteration_step += 1
```

**이 코드에는 아래와 같은 조건 분기가 없음:**

```python
# 이렇게 했어야 함
if not tool_calls:
    final_answer += response + "\n"
```

GPT-4에서는 tool_call 턴의 content가 null이라 빈 문자열이 누적되어 문제가 없었지만, gpt-oss에서는 가짜 답변이 누적됨.

### 최종 결과

```
final_answer = "가짜 답변 JSON\n실제 답변 JSON"
                ↑ 턴 1 (tool_call)     ↑ 턴 2 (최종)
```

→ JSON이 2개 붙어서 출력되는 현상의 정확한 원인.

---

## 5. 문제 2: 병렬 tool call

### 현상

curl 테스트 결과 `tool_calls` 배열에 2개의 항목이 포함:

```json
"tool_calls": [
  {"function": {"name": "sql_executor_tool", "arguments": "{\"query\": \"SELECT ... MySQL 문법\"}"}},
  {"function": {"name": "sql_executor_tool", "arguments": "{\"query\": \"SELECT ... SQLite 문법\"}"}}
]
```

- Dify를 거치지 않은 raw API 응답에서도 2개 → **모델 자체의 행동**
- 동일 쿼리를 다른 SQL 방언(MySQL/SQLite/PostgreSQL)으로 보내거나, 완전 동일 쿼리를 중복 발행
- `reasoning_content`에서도 두 번 작성하는 패턴이 보임
- `tool_choice: "required"` 적용해도 변화 없음

### 병렬 호출의 결정 주체

**모델이 추론 과정에서 스스로 결정**한다. 명시적인 규칙이 아니라 학습된 패턴에 따라 "여러 도구를 동시에 쓰는 게 효율적이다"고 판단하면 여러 개를 생성.

chat_template.jinja 286줄에도 설계 의도가 드러남:

```jinja
{#- We assume max 1 tool call per message, and so we infer the tool call name #}
{%- set tool_call = message.tool_calls[0] %}
```

template 설계자가 **"메시지당 tool call은 최대 1개"를 전제**하고 있으나, 모델은 이를 따르지 않음.

### curl 테스트에서 관찰된 패턴

| 테스트 | tool_calls 개수 | reasoning_content에서 드러난 의도 |
|--------|----------------|--------------------------------|
| 기본 (파라미터 없음) | 2개 (MySQL + SQLite 문법) | SQL 방언을 몰라서 두 가지로 시도 |
| `tool_choice: "required"` | 2개 (동일 쿼리 중복) | 같은 쿼리를 두 번 작성 |
| **`parallel_tool_calls: false`** | **1개** | 단일 쿼리만 생성 |

### 병렬 호출이 유용한 경우 vs 불필요한 경우

```
# 유용한 경우 — 독립적인 데이터를 동시에 조회
사용자: "미국과 일본의 올해 매출을 비교해줘"
→ tool_calls: [미국 매출 쿼리, 일본 매출 쿼리]  ← 합리적

# 불필요한 경우 — 불확실성으로 인한 중복
사용자: "올해 매출 알려줘"
→ tool_calls: [MySQL 쿼리, SQLite 쿼리]  ← SQL 방언을 몰라서 시도
```

gpt-oss의 병렬 호출은 주로 **"효율적인 병렬 처리"가 아니라 "불확실성 때문에 여러 변형을 시도"**하는 패턴.

### `parallel_tool_calls: false` — 해결 확인

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/install_file_backup/tessinu/gpt-oss-120b",
    "messages": [
      {"role": "user", "content": "올해 월별 매출을 알려줘"}
    ],
    "parallel_tool_calls": false,
    "tools": [...]
  }' 2>/dev/null | python3 -m json.tool
```

결과: `tool_calls` 배열에 **1개만** 포함. 병렬 호출 문제 해결 확인.

### Dify에서의 적용

Dify UI에는 `parallel_tool_calls` 설정이 **노출되어 있지 않음**.

| 방법 | 가능 여부 |
|------|----------|
| Agent 노드 UI | 없음 |
| 모델 provider YAML | 없음 |
| 워크플로우 DSL (YAML export) | 없음 |
| 환경변수 | 없음 |
| **Dify 소스 수정** | 가능 (LLM provider 코드에서 강제 설정) |
| **vLLM 서버단 proxy/설정** | 가능 (Dify를 안 건드리고 해결) |
| **프롬프트 가이드** | 보조적 완화 가능 |

**Dify 소스 수정 시 위치:**

```
api/core/model_runtime/model_providers/openai/llm/llm.py
→ tools가 있을 때 extra_params["parallel_tool_calls"] = False 추가
```

이 파일은 Dify에서 **OpenAI 호환 API를 호출하는 공통 코드**. 모든 OpenAI 호환 모델 요청이 이 파일을 거쳐 나가기 때문에, 여기를 수정하면 **모든 에이전트에 일괄 적용**됨 (특정 에이전트만 적용 불가).

```
Dify 에이전트 A (매출 분석) ─┐
Dify 에이전트 B (전자결재)  ─┤→ openai/llm/llm.py → vLLM API
Dify 에이전트 C (기타)     ─┘
```

에이전트별로 다르게 설정하려면, Agent 노드 UI에 `parallel_tool_calls` 옵션이 추가되어야 함 (temperature, max_tokens처럼). 현재 Dify에는 이 옵션이 없으므로 feature request 또는 PR이 필요.

**프롬프트 가이드 예시:**

```
도구는 한 번에 하나만 호출하세요. 결과를 받은 후 다음 도구를 호출하세요.
SQL은 반드시 PostgreSQL 문법만 사용하세요.
```

> SQL 방언을 프롬프트에 명시하면 "불확실성으로 인한 중복 시도" 패턴이 줄어들 수 있음.

---

## 6. chat_template.jinja 분석

### 공식 버전과 로컬 비교

- HuggingFace 공식: `https://huggingface.co/openai/gpt-oss-120b/blob/main/chat_template.jinja`
- 서버 로컬: `/install_file_backup/tessinu/gpt-oss-120b/chat_template.jinja`
- 결과: **기능적으로 동일** (developer_message 뒤 `\n\n` 줄바꿈 하나 차이)

### 흥미로운 설계: 292-293줄

```jinja
{%- elif message.content and not future_final_message.found %}
    {{- "<|start|>assistant<|channel|>analysis<|message|>" + message.content + "<|end|>" }}
```

tool_calls가 있는 assistant 메시지를 다음 턴에 다시 모델에게 보낼 때, **content를 final이 아닌 analysis 채널로 변환**함.

이는 template 설계자가 "tool_call 턴의 content는 사고(analysis)이지 최종 답변(final)이 아니다"라고 인식하고 있었음을 의미. 모델이 final 채널까지 생성하는 문제를 **template 레벨에서 우회 처리**한 것.

---

## 7. reasoning_parser 상태

| 항목 | 상태 |
|------|------|
| `--reasoning-parser openai_gptoss` | 작동 중 (자동 또는 수동 적용) |
| analysis → `reasoning_content` 분리 | 정상 |
| commentary → `tool_calls` 파싱 | 정상 |
| final → `content` 매핑 | 정상 |

reasoning_parser는 **모델이 생성한 출력을 채널별로 분류**하는 역할. 모델이 final 채널을 생성하는 것 자체를 막을 수는 없음. parser는 정상 작동하고 있으며, 문제는 모델이 tool call 시에도 final 채널을 생성하는 행동에 있음.

---

## 8. 해결 방안

### 적용 가능한 레이어

| 레이어 | 방법 | 난이도 | 비고 |
|--------|------|--------|------|
| 모델 | 재학습/fine-tuning | 불가 | 모델 제작사만 가능 |
| vLLM | tool_calls 감지 시 content를 null로 처리 | 중 | vLLM 소스 수정 필요 |
| Dify 소스 | tool_calls 있을 때 content 누적 안 함 | 중 | 업데이트 시 깨질 수 있음 |
| **Dify 코드 노드** | **last-JSON 추출로 우회** | **하** | **현실적 선택** |

### 코드 노드 방어 로직 (last-JSON 추출)

```python
import json
import re

def main(agent_text: str) -> dict:
    text = re.sub(r'<think>.*?</think>', '', agent_text, flags=re.DOTALL).strip()

    # 모든 top-level JSON 블록 추출, 마지막 것 사용 (실제 데이터)
    json_blocks = []
    brace_depth = 0
    start = -1
    for i, ch in enumerate(text):
        if ch == '{':
            if brace_depth == 0:
                start = i
            brace_depth += 1
        elif ch == '}':
            brace_depth -= 1
            if brace_depth == 0 and start != -1:
                candidate = text[start:i+1]
                try:
                    json.loads(candidate)
                    json_blocks.append(candidate)
                except json.JSONDecodeError:
                    pass
                start = -1

    if not json_blocks:
        return {
            "json": agent_text,
            "analysis_result": "분석 결과를 처리하는 중 오류가 발생했습니다.",
            "chart": False, "chartType": "", "chartData": "{}",
            "insights": "", "further_analysis": "",
            "email": False, "email_to": "", "email_subject": "",
            "markdown": "분석 결과를 처리하는 중 오류가 발생했습니다.",
        }

    data = json.loads(json_blocks[-1])  # 마지막 JSON = 실제 데이터 기반 응답

    if "output" in data and isinstance(data["output"], dict):
        data = data["output"]
    data.pop("think", None)

    # Markdown 생성
    md_parts = []
    if data.get("analysis_result"):
        md_parts.append(data["analysis_result"])
    if data.get("insights"):
        md_parts.append("\n**인사이트**\n" + data["insights"])
    if data.get("further_analysis"):
        md_parts.append("\n**추가 분석 제안**\n" + data["further_analysis"])

    return {
        "json": json.dumps(data, ensure_ascii=False),
        "analysis_result": data.get("analysis_result", ""),
        "chart": data.get("chart", False),
        "chartType": data.get("chartType", ""),
        "chartData": json.dumps(data.get("chartData", {}), ensure_ascii=False),
        "insights": data.get("insights", ""),
        "further_analysis": data.get("further_analysis", ""),
        "email": data.get("email", False),
        "email_to": data.get("email_to", ""),
        "email_subject": data.get("email_subject", ""),
        "markdown": "\n".join(md_parts),
    }
```

원리: `final_answer`에 "가짜JSON\n실제JSON"이 들어오면, 모든 JSON 블록을 파싱한 후 **마지막 것만 사용**. 마지막 JSON이 tool 결과를 기반으로 생성된 실제 답변.

---

## 9. 프롬프트 기반 해결 시도 (실패)

### Tool Call Discipline 프롬프트

전자결재 에이전트에서는 효과적이었던 방식:

```
[Tool Call Discipline]
- tool 호출 전에는 10단어 이내의 짧은 대기 멘트만 출력
- JSON 형식으로 작성하지 않을 것
- 최종 응답은 반드시 tool result 수신 후에만 작성
```

**매출 분석 에이전트에서는 실패**: `[Output JSON Format]` 지시("반드시 아래 JSON 형태로만 최종 응답하세요")가 Tool Call Discipline보다 우선하여, 모델의 analysis 채널이 여전히 JSON으로 가짜 답변을 생성.

→ **JSON 출력을 요구하는 에이전트에서는 프롬프트만으로 해결 불가**. 코드 노드 방어가 필수.

---

## 10. generation_config.json과 `<|call|>` 토큰

### eos_token_id란

**End Of Sequence token ID** — 모델이 이 토큰을 생성하면 "생성 끝" 신호로 멈춤.

현재 서버의 설정:
```json
{
  "eos_token_id": [200002, 199999],  // <|end|>, <|endoftext|>
  // 200012 (<|call|>)은 없음
}
```

### `<|call|>` 토큰(200012)

gpt-oss에서 **tool call 완료를 표시하는 토큰**. chat_template에서:

```
<|start|>assistant to=functions.sql_executor_tool<|channel|>commentary json<|message|>
{"query": "SELECT ..."}
<|call|>               ← 여기가 200012
```

### `<|call|>`이 eos에 없는 것의 영향

`<|call|>`이 `eos_token_id`에 없기 때문에, 모델이 tool call을 완성한 후에도 멈추지 않고 계속 생성:

```
<|call|>이 EOS에 있으면:
  tool call 생성 → <|call|> → 멈춤 (tool call 1개, final 채널 없음)

<|call|>이 EOS에 없으면 (현재):
  tool call 생성 → <|call|> → 안 멈추고 계속 생성
  → 두 번째 tool call (병렬 호출)
  → final 채널 (가짜 답변)
  → <|end|>(200002) 만나서야 멈춤
```

이것이 **병렬 호출과 가짜 답변, 두 문제 모두의 근본 원인**.

### `stop_token_ids: [200012]` 테스트 — 실패

API 요청에서 `<|call|>`을 stop token으로 지정해봤으나:

```json
{
  "tool_calls": [],        // 빈 배열 — tool call 자체가 안 됨
  "finish_reason": "stop"  // tool_calls가 아닌 일반 종료
}
```

모델이 `<|call|>` 토큰을 생성하려다 stop 되어서 tool call이 완성되지 않음. **tool calling 자체가 망가짐**.

### 딜레마

| `<|call|>` 위치 | 병렬 호출 | 가짜 답변 | tool calling |
|-----------------|----------|----------|-------------|
| EOS에 없음 (현재) | 발생 | 발생 | **정상** |
| EOS에 있음 | 없음 | 없음 | **망가짐** |
| stop_token_ids로 지정 | - | - | **망가짐** |

→ `<|call|>` 토큰 레벨에서는 해결 불가. `parallel_tool_calls: false` + 코드 노드 방어가 최선.

---

## 11. 20b vs 120b 비교

### chat_template.jinja

- 20b, 120b **구조 동일** — 같은 3채널 시스템, 같은 버그
- 공식 HuggingFace에서 확인: [openai/gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b)

### tool calling 동작 차이

- 20b와 120b 사이에 tool calling 관련 **문서화된 동작 차이 없음**
- 둘 다 동일한 tool calling 문제가 여러 provider(vLLM, NVIDIA NIM, Ollama, llama.cpp)에서 보고됨
- Ollama에서 20b가 단일 호출만 했던 건 **Ollama의 tool call 처리 방식** 차이일 가능성

### chat_template의 알려진 버그

| 버그 | 내용 |
|------|------|
| content → analysis 오분류 | tool call 전 content를 commentary(사용자 메시지)가 아닌 analysis(CoT)로 처리 |
| `<\|constrain\|>` 토큰 누락 | JSON 제약 조건에 special token 대신 plain text 사용 |
| 이전 턴 CoT 미정리 | 불필요한 토큰 낭비 |

### 다른 provider에서의 동일 문제 보고

- [vLLM #22337](https://github.com/vllm-project/vllm/issues/22337): tool call 데이터가 `content`에 텍스트로 출력
- [NVIDIA NIM 포럼](https://forums.developer.nvidia.com/t/tool-calling-gpt-oss-20b-and-120b/341611): 20b, 120b 둘 다 동일 문제
- [Ollama #11704](https://github.com/ollama/ollama/issues/11704): gpt-oss-120b의 malformed tool calls
- OpenAI는 아직 공식 대응 없음

---

## 12. 핵심 결론

1. **reasoning_parser는 정상 작동** — 채널별 분리가 올바르게 이루어지고 있음
2. **`<|call|>` 토큰이 eos_token_id에 없어서 모델이 tool call 후 멈추지 않는 것이 근본 원인** — 병렬 호출과 가짜 답변 모두 이것에서 비롯
3. **Dify가 모든 round의 content를 무조건 누적하는 것이 확대 원인** — GPT-4 전제 설계로, content가 null이 아닌 모델과 호환성 문제 발생
4. **`<|call|>`을 stop token으로 넣으면 tool calling 자체가 망가짐** — 토큰 레벨 해결 불가
5. **gpt-oss 20b/120b 모두 동일한 문제** — 모델 크기와 무관, 여러 provider에서 보고됨
6. **현실적 해결: `parallel_tool_calls: false` + 코드 노드 last-JSON 추출**

---

## 참고 자료

- Dify FC Agent Runner: [fc_agent_runner.py](https://github.com/langgenius/dify/blob/main/api/core/agent/fc_agent_runner.py)
- Dify Plugin FC Strategy: [function_calling.py](https://github.com/langgenius/dify-official-plugins/blob/main/agent-strategies/cot_agent/strategies/function_calling.py)
- gpt-oss chat_template: [HuggingFace 120b](https://huggingface.co/openai/gpt-oss-120b/blob/main/chat_template.jinja) / [HuggingFace 20b](https://huggingface.co/openai/gpt-oss-20b)
- Dify 중복 출력 관련 이슈: [GitHub Issue #4705](https://github.com/langgenius/dify/issues/4705)
- vLLM tool call 이슈: [GitHub Issue #22337](https://github.com/vllm-project/vllm/issues/22337)
- NVIDIA NIM tool calling: [포럼 스레드](https://forums.developer.nvidia.com/t/tool-calling-gpt-oss-20b-and-120b/341611)
