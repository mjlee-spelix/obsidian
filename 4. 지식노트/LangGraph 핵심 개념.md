---
tags:
  - langgraph
  - AI-Agent
  - 개발
  - LLM
created: 2026-06-30
---

# LangGraph 핵심 개념

> 대책서 도우미 LangGraph 설계 문서를 이해하기 위한 기초 개념 정리. 예시는 모두 그 설계에서 가져옴.

## 한 줄 요약
LangGraph = **상태(State)를 들고 노드(Node) 사이를 엣지(Edge)로 이동하는 그래프**. 워크플로우를 "함수 호출 순서"가 아니라 "그래프"로 표현하는 도구. Dify의 노드·연결선과 1:1로 대응된다.

## 1. State (공유 상태)
모든 노드가 **함께 읽고 쓰는 하나의 딕셔너리**. 노드 간에 값을 인자로 넘기는 게 아니라, 이 State에 쓰면 다음 노드가 읽는다.

```python
class RecoState(TypedDict, total=False):
    payload: dict
    ok: bool
    step: Literal["cause", "countermeasure"]
    internal_items: list[dict]
    result: dict
```

- `TypedDict` = 키 이름과 타입을 정의한 딕셔너리(런타임엔 그냥 dict).
- `total=False` = 모든 키가 필수는 아님(노드가 점점 채워나감).
- **Dify 대응**: Dify에서 각 노드 출력을 `value_selector`로 가져다 쓰던 것 → LangGraph에선 그냥 State 키를 읽으면 됨.

## 2. Node (노드)
**State를 받아 → 일부를 갱신해 반환하는 함수**. 반환한 dict가 State에 병합된다.

```python
def parse_input(state: RecoState):
    payload = state["payload"]
    # ...검증...
    return {"ok": True, "step": "cause", "internal_count": 5}  # 이 키들만 갱신
```

- 반환하지 않은 키는 그대로 유지된다(전체를 반환할 필요 없음).
- **Dify 대응**: Dify의 "코드 노드 / LLM 노드 / HTTP 노드" 하나하나가 LangGraph 노드 함수 하나.

## 3. Edge (엣지)
노드 간 **고정 연결선**. "A 끝나면 무조건 B로."

```python
g.add_edge("kr_retrieve", "llm_extract")   # KR → 추출 (항상)
g.add_edge(START, "parse_input")           # 시작점
```

- `START` / `END` = 그래프의 시작·끝을 나타내는 특수 노드.
- **Dify 대응**: 노드를 잇는 화살표.

## 4. Conditional Edge (조건부 엣지) — 분기
**함수의 반환값으로 다음에 갈 노드를 고르는** 엣지. Dify의 IF/ELSE에 해당.

```python
def route_after_parse(state):
    if not state["ok"]:
        return "error_end"                      # 검증 실패 → 에러 종료
    return ["kr_retrieve", "build_external_request"]  # 정상 → 둘 다 (병렬!)

g.add_conditional_edges(
    "parse_input", route_after_parse,
    ["error_end", "kr_retrieve", "build_external_request"],  # 갈 수 있는 후보 목록
)
```

- 라우터 함수는 **노드 이름(문자열)** 을 반환한다.
- **리스트를 반환하면 여러 노드를 동시에 실행** = 다음 개념(병렬)으로 이어짐.
- **Dify 대응**: 설계 문서의 "검증 게이트", "단계 분기" IF/ELSE.

## 5. Fan-out (병렬 분기)
조건부 엣지가 **리스트를 반환**하면 그 노드들이 **동시에** 출발한다. 설계에서 내부 RAG 경로와 외부 LLM 경로가 여기서 갈라짐.

```
parse_input ─┬─→ kr_retrieve → llm_extract → format_internal ─┐
             └─→ build_external_request → call_external_llm  ─┴─→ merge_sort
```

## 6. Fan-in (병합) & Superstep
한 노드에 **여러 화살표가 들어오면**, LangGraph는 **선행 노드들이 전부 끝날 때까지 기다렸다가** 그 노드를 **한 번만** 실행한다.

```python
g.add_edge("format_internal", "merge_sort")    # 내부 경로 도착
g.add_edge("call_external_llm", "merge_sort")   # 외부 경로 도착
# → merge_sort는 둘 다 끝나야 1회 실행
```

- **Superstep(슈퍼스텝)**: LangGraph 실행 단위. "같은 단계에서 동시에 돌 수 있는 노드들"을 한 묶음으로 실행하고, 다 끝나면 다음 묶음으로 넘어간다. 내부 경로가 3노드라 더 길어도, 병합 노드가 알아서 외부 경로 완료까지 대기한다.
- **Dify 대응**: 두 경로가 "병합·정렬" 노드로 모이는 지점.

## 7. Reducer (상태 병합 규칙) ⚠️ 병렬에서 중요
두 노드가 **동시에 같은 State 키**를 쓰면 충돌한다. 기본 동작은 "덮어쓰기"라 한쪽 값이 사라질 수 있다.

- **해결 1 (설계에서 채택)**: 병렬 경로가 **서로 다른 키**를 쓰게 한다.
  - 내부 경로 → `internal_items`, 외부 경로 → `external_response`. 충돌 없음.
- **해결 2**: 합쳐야 할 땐 reducer를 단다.
  ```python
  from typing import Annotated
  import operator
  items: Annotated[list, operator.add]   # 양쪽 리스트를 이어붙임
  ```

이게 설계 문서 "주의점 1. 병렬 reducer" 항목의 의미다.

## 8. compile() & invoke()
그래프 정의를 **실행 가능한 객체로 컴파일**한 뒤 호출한다.

```python
app = g.compile()
result = app.invoke({"payload": {...}})   # 초기 State 넣고 실행 → 최종 State 반환
```

## 9. Checkpointer (체크포인터) — 이번엔 일부러 안 씀
LangGraph는 State를 저장소(메모리·DB)에 **체크포인트**로 남겨, 대화를 이어가거나 중단 후 재개할 수 있다(`MemorySaver` 등).

- **그런데 이 설계는 무상태(stateless)** 가 핵심이라 **checkpointer를 쓰지 않는다.**
- 세션·누적·저장은 프론트가 소유하고, "더 찾기"는 `previous_response_id`만 payload로 주고받는다.
- 즉 "LangGraph의 기억 기능"과 "OpenAI Responses API의 대화 이어가기"는 별개 — 후자만 쓴다.

## 10. with_structured_output (구조화 출력)
LLM이 **자유 텍스트가 아니라 정해진 JSON 스키마**로만 답하게 강제한다. 설계의 `llm_extract` 노드에서 `{items:[{text, source_doc}]}` 형태를 보장하는 장치.

```python
structured_llm = llm.with_structured_output(schema)
out = structured_llm.invoke(prompt)   # 항상 schema 형태의 객체
```

- **Dify 대응**: LLM 노드의 `structured_output_enabled: true` + schema.

---

## Dify ↔ LangGraph 용어 대응표

| Dify | LangGraph |
|------|-----------|
| 노드(코드/LLM/HTTP) | Node 함수 |
| 노드 잇는 화살표 | `add_edge` |
| IF/ELSE | `add_conditional_edges` |
| value_selector로 출력 참조 | State 키 읽기 |
| LLM structured_output | `with_structured_output` |
| 병렬 경로 합류 | fan-in (여러 엣지 → 한 노드) |
| (해당 없음 — 무상태) | checkpointer 미사용 |

## 관련 노트
- [[3. 프로젝트/서연이화/과제5/langgraph 구현 설계.md]]
