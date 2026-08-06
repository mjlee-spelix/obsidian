---
tags: [지식, dify, api, sse, streaming]
date: 2026-03-26
---
# Dify - SSE vs Blocking 응답 모드

## 핵심
- Dify API의 `response_mode`는 `blocking`과 `streaming` 두 가지가 있음
- `blocking`은 처리 완료 후 JSON을 한 번에 반환, `streaming`은 SSE로 토큰 단위 실시간 전송
- **에이전트 chatflow는 `blocking`으로 요청해도 SSE로 응답함** (도구 호출 등 처리 시간이 길어서)
- `blocking` + 에이전트 조합 시 `response.text()`가 스트림 종료까지 무한 대기하므로, **`streaming` 모드 + 스트림 리더를 사용해야 함**

## 상세

### Blocking 모드
서버가 처리를 완전히 끝낸 뒤 응답 전체를 한 번에 반환하는 방식.

```
클라이언트 → 요청 → [서버 처리 대기] → JSON 응답 한 번에 수신
```

```json
{
  "answer": "분석 결과입니다",
  "conversation_id": "abc-123"
}
```

- 구현이 단순함 (`fetch` → `response.json()`)
- 처리 시간이 길면 타임아웃 위험
- ⚠️ 에이전트 chatflow에서는 사용 불가 (`response.text()`가 영원히 블로킹됨)

### Streaming 모드 (SSE)
**SSE (Server-Sent Events)**: 서버가 클라이언트에게 텍스트 스트림으로 데이터를 실시간 전송하는 HTTP 프로토콜.

```
클라이언트 → 요청 → 즉시 "결" → "과" → "입니다" → ... → 완료
```

- ChatGPT 타이핑 효과처럼 토큰 단위로 조각 전송
- `event:` 줄이 SSE 이벤트 라벨, `data:` 줄이 JSON 페이로드
- 빈 줄(`\n\n`)이 이벤트 구분자

### Dify Chatflow의 실제 SSE 형식

Dify chatflow에서는 SSE `event:` 줄이 항상 `ping`이고, **실제 이벤트 타입은 JSON 내부 `data.event` 필드**에 있다.

```
event: ping
data: {"event": "workflow_started", "workflow_run_id": "...", ...}

event: ping
data: {"event": "node_started", "data": {"title": "에이전트", ...}, ...}

event: ping
data: {"event": "agent_log", "data": {"label": "도구 호출", ...}, ...}

event: ping
data: {"event": "node_finished", "data": {"title": "에이전트", "outputs": {...}}, ...}

event: ping
data: {"event": "message", "answer": "{...JSON 문자열...}", "conversation_id": "..."}

event: ping
data: {"event": "workflow_finished", "data": {"outputs": {"answer": "..."}, "total_tokens": 2100, ...}}
```

### 이벤트 타입 (data.event 기준)

| 이벤트 타입 | 설명 | 주요 필드 |
|---|---|---|
| `workflow_started` | 워크플로우 시작 | workflow_run_id |
| `node_started` | 노드 실행 시작 | data.title (노드 이름) |
| `node_finished` | 노드 실행 완료 | data.title, data.outputs |
| `agent_log` | 에이전트 내부 동작 (도구 호출, 추론) | data.label, data.text |
| `message` | ★ **답변 노드 출력** (answer 포함) | answer, conversation_id |
| `workflow_finished` | 워크플로우 종료 | data.outputs, data.total_tokens, data.elapsed_time |

### 실행 순서 예시

```
workflow_started
node_started   node=시작
node_finished  node=시작
node_started   node=에이전트
agent_log      (도구 호출, SQL 실행 등 여러 번)
node_finished  node=에이전트
node_started   node=IF/ELSE
message        answer=yes(694)    ← ★ 답변 텍스트 (에이전트가 생성한 answer가 flush됨)
node_finished  node=IF/ELSE
node_started   node=답변
node_finished  node=답변           ← 답변 노드는 패스스루 (자체 answer 없음)
workflow_finished                  ← 최종 outputs 포함
```

> **주의**: `message` 이벤트는 답변 노드(`node_started node=답변`) 이전에 발생한다.
> 답변 노드는 패스스루 역할이며, 실제 answer는 에이전트 노드가 생성하고 `message` 이벤트로 먼저 flush된다.

### 비교

| 구분            | Blocking          | Streaming (SSE)    |
| ------------- | ----------------- | ------------------ |
| 응답 방식         | 완료 후 JSON 한 번     | 토큰 단위 실시간          |
| 구현 난이도        | 낮음 (`JSON.parse`) | 높음 (SSE 스트림 리더 필요) |
| 사용자 경험        | 긴 대기 후 한 번에 표시    | 실시간 타이핑 효과         |
| 타임아웃          | 긴 처리 시 위험         | ping으로 연결 유지       |
| 에이전트 chatflow | ❌ 사용 불가 (무한 블로킹)  | ✅ 필수               |

### SSE에서 확인 가능/불가능한 정보

| 확인 가능 | 확인 불가 (Dify 웹 UI 필요) |
|---|---|
| 실행 흐름 (노드 순서) | 각 노드의 입력값 상세 |
| 노드 이름, 타입, ID | LLM 프롬프트/응답 원문 |
| 노드별 출력(outputs) | SQL 쿼리 실행 결과 상세 |
| 에이전트 로그 (agent_log) | 에러 스택트레이스 |
| 최종 결과, 토큰 수, 소요 시간 | |

## 구현: SSE 스트림 리더

`response.text()`는 스트림이 닫힐 때까지 블로킹하므로, `ReadableStream`을 줄 단위로 읽어야 한다.

```typescript
async function readDifyStream(response: Response) {
  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = '';
  let answer = '';
  let conversationId = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    while (buffer.includes('\n')) {
      const idx = buffer.indexOf('\n');
      const line = buffer.slice(0, idx).trim();
      buffer = buffer.slice(idx + 1);

      if (line.startsWith('data: ')) {
        const data = JSON.parse(line.slice(6));
        const eventType = data.event;  // ← SSE event: 줄이 아닌 JSON 내부 필드

        if (data.answer) answer += data.answer;
        if (data.conversation_id) conversationId = data.conversation_id;

        // workflow_finished에서 스트림 종료
        if (eventType === 'workflow_finished') {
          // 폴백: data.data.outputs.answer
          if (!answer && data.data?.outputs?.answer) {
            answer = data.data.outputs.answer;
          }
          reader.cancel();
          return { answer, conversation_id: conversationId };
        }
      }
    }
  }
  return { answer, conversation_id: conversationId };
}
```

### 삽질 기록

1. **`blocking` 모드 + `JSON.parse(responseText)`** → `SyntaxError: "event: ping"은 유효한 JSON이 아님`
2. **`blocking` 모드 + `response.text()`** → 스트림이 안 닫혀서 5분간 무한 대기 후 타임아웃
3. **`streaming` 모드 + SSE `event:` 줄 기준 파싱** → 모든 이벤트가 `event: ping`이라 answer 수집 실패
4. **`streaming` 모드 + `data.event` 기준 파싱** → ✅ 성공 (`message` 이벤트에서 answer 추출)

## 관련 노트
- [[Dify - Sandbox 개념 및 matplotlib 설치]]
- [[Dify - 코드 노드 시각화 구현]]
