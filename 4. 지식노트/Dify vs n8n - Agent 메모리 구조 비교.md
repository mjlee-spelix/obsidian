---
tags: [지식, dify, n8n, LLM, 에이전트, 메모리, 비교]
date: 2026-04-16
---
# Dify vs n8n — Agent 메모리 구조 비교

## 핵심
- Dify agent memory(window)는 **user 텍스트 + assistant 최종 텍스트만** 저장. tool_result(raw_data 등)는 다음 턴에 전달되지 않음
- n8n agent memory는 **tool_call + tool_result까지 전부** LangChain 메시지 형식으로 저장
- 이 차이가 "후속 차트 요청 시 DB 재조회" 같은 동작 차이를 만들어냄

---

## Dify

### 구현 위치
`api/core/memory/token_buffer_memory.py` — `get_history_prompt_messages()`

### 저장 내용
DB `Message` 테이블의 두 필드만 읽음:
- `message.query` → `UserPromptMessage`
- `message.answer` → `AssistantPromptMessage`

```python
for message in messages:
    prompt_messages.append(UserPromptMessage(content=message.query))
    prompt_messages.append(AssistantPromptMessage(content=message.answer))
```

### window.size 동작
`agent_node.py`에서 `message_limit=window.size`로 DB 조회 개수를 제한.
이후 토큰 초과 시 오래된 메시지부터 추가 프루닝.

### 결과
한 턴 내부의 nl2sql tool_result(raw_data JSON 배열)는 다음 턴 LLM에게 전달되지 않음.
chat history에는 agent가 생성한 마크다운 텍스트만 남음.

---

## n8n

### 구현 위치
`packages/@n8n/nodes-langchain/utils/agent-execution/memoryManagement.ts` — `saveToMemory()` / `loadMemory()`

### 저장 내용
한 턴이 끝나면 **4종류의 메시지**를 순서대로 저장:

```typescript
messages.push(new HumanMessage(input))             // 사용자 질문
messages.push(new AIMessage({ tool_calls: [...] })) // LLM의 도구 호출 결정 + 인자
messages.push(new ToolMessage({ content: 결과 }))   // 도구 실행 결과 (raw_data 등)
messages.push(new AIMessage(output))               // 최종 텍스트 응답
```

`buildMessagesFromSteps()`가 각 tool call step을 AIMessage + ToolMessage 쌍으로 변환.

### 토큰 기반 윈도우
`maxTokensFromMemory` 설정으로 제어. 토큰 초과 시 `trimMessages(strategy: 'last')`로 최근 메시지부터 유지.
잘린 경우 `cleanupOrphanedMessages()`가 불완전한 tool_call 시퀀스를 정리.

### 결과
후속 턴 LLM이 이전 도구 결과를 ToolMessage로 그대로 받음.
"차트 생성해줘" 같은 후속 요청에서 DB 재조회 없이 이전 raw_data 재사용 가능.

---

## 비교 표

| 항목 | Dify | n8n |
|---|---|---|
| 저장 단위 | user 텍스트 + assistant 텍스트 | HumanMessage + AIMessage(tool_calls) + ToolMessage + AIMessage |
| tool_result 포함 | **No** | **Yes** |
| window 제어 | 메시지 개수 (window.size) | 토큰 수 (maxTokensFromMemory) |
| 후속 요청 시 이전 데이터 | 마크다운 텍스트만 참조 가능 | 원본 JSON 데이터 참조 가능 |
| 저장 위치 | DB Message 테이블 (query/answer 컬럼) | LangChain BaseChatMemory (인메모리 or 외부 store) |

---

## 실무 영향

### Dify에서의 문제
"올해 월별 매출 분석해줘" → "차트 생성해줘" 시나리오:
- 1턴 raw_data JSON이 chat history에 없음
- agent instruction에 "후속 요청 시 nl2sql 재호출 금지"가 있어도 raw_data를 찾지 못해 결국 재호출

### Dify 대응 방법
1. **instruction 분기**: 신규 차트(nl2sql 먼저) vs 후속 차트(chat history 마크다운 표 재사용) 명시적으로 구분
2. **chart workflow LLM 유연화**: raw_data 입력으로 JSON 배열뿐 아니라 마크다운 표도 허용하도록 시스템 프롬프트 수정

### n8n에서는
ToolMessage로 raw_data가 자동 보존되므로 별도 대응 불필요.
단, 대용량 데이터는 ToolMessage가 컨텍스트를 많이 차지함 → maxTokensFromMemory 튜닝 필요.
