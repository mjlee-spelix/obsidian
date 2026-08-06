# A — Dify 토폴로지 보존형 LangGraph

학습용 베이스라인이다. 원인/대책 노드군을 물리적으로 분리하고 외부 호출도
`요청 바디 → raw HTTP → Responses API 언랩`으로 나눴다.

- 활성 노드: 20개
- 시각적 방향 엣지: 25개(조건 분기 후보, wait-all의 두 입력, START/END 포함)
- orphan 표시: 2개(Dify id `2003000000005`, `2003000000009`)
- 상태 저장/checkpointer: 사용하지 않음
- 비밀값: 기본 raw HTTP transport가 `OPENAI_API_KEY`만 읽음

관리형 Dify 지식검색과 내부 LLM은 `Services`로 주입한다. 주입하지 않으면 내부
후보는 비어 있으며, 외부 호출 실패도 Dify의 default-value 전략처럼 빈 응답으로
폴백한다.

```python
from measure_langgraph_literal import build_graph

app = build_graph()
state = app.invoke({"payload": {"step": "cause", "phenomenon": "도어 단차"}})
print(state["result"])
```

실제 LangGraph가 설치되어 있으면 `StateGraph`를 컴파일한다. 설치되지 않은 학습/
테스트 환경에서는 동일 노드 함수를 쓰는 로컬 실행 어댑터가 자동 사용된다.

테스트:

```powershell
python -m unittest discover -s measure-langgraph/literal/tests -v
```

