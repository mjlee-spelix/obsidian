# 하이브리드 LangGraph 구현

Dify 코드 노드의 입력 검증, 내부 후보 정형화, 병합·정렬 본체를 보존하고
검색·구조화 추출·OpenAI Responses 호출은 주입 가능한 네이티브 포트로 바꾼 버전입니다.

```powershell
cd measure-langgraph/hybrid
python -m pip install -r requirements.txt
python -m unittest discover -s tests -v
```

`build_graph(Services(...))`에 `Retriever`, `InternalExtractor`, `ResponsesClient`
구현을 전달합니다. 실제 외부 호출용 `OpenAIResponsesClient`는
`OPENAI_API_KEY`만 읽습니다. 체크포인터는 사용하지 않으며
`previous_response_id`는 요청 payload와 결과 `meta.responseId` 사이에서만 왕복합니다.

현재 사내 bge-m3-ko 벡터 저장소와 gemma/vLLM 엔드포인트는 확정되지 않았으므로
각 포트의 운영 어댑터는 별도 연결이 필요합니다.
