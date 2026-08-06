# Native LangGraph 구현

설계서 §3~§5의 통합 토폴로지를 전 노드 재작성한 버전이다. 그래프는
checkpointer를 사용하지 않으며, 내부 검색·추출·외부 호출은 `ports.py`의
인터페이스로 주입한다.

```python
from native import WorkflowDependencies, build_graph

app = build_graph(WorkflowDependencies(retriever, extractor, external_client))
result = app.invoke({"payload": payload})["result"]
```

## 실제 API 어댑터

다음 운영 어댑터가 포함되어 있다.

- `dify_retriever.DifyKnowledgeRetriever`: Dify Knowledge API로 유사 문서 검색
- `openai_extractor.OpenAIInternalExtractor`: 검색 문서만 근거로 내부 원인/대책 추출
- `clients.OpenAIResponsesClient`: OpenAI만 사용해 외부 원인/대책 생성

비밀값은 소스에 하드코딩하지 않고 환경변수에서 읽는다. `smoke_live`는
`native/.env`를 자동으로 로드하며, 이미 설정된 PowerShell 환경변수가 있으면 그 값을 우선한다.
`native/.env`는 `.gitignore`에 포함되어 있다.

```powershell
Copy-Item native/.env.example native/.env
# native/.env의 placeholder를 실제 값으로 수정
```

또는 현재 PowerShell 세션에 직접 설정할 수 있다.

```powershell
$env:DIFY_API_BASE_URL="https://api.dify.ai/v1"
$env:DIFY_DATASET_ID="지식베이스-ID"
$env:DIFY_KNOWLEDGE_API_KEY="지식베이스-API-KEY"
$env:OPENAI_API_KEY="OpenAI-API-KEY"

# 선택값. 지정하지 않으면 gpt-5-nano
$env:OPENAI_INTERNAL_MODEL="gpt-5-nano"
```

self-hosted Dify는 `DIFY_API_BASE_URL`을 해당 서버의 `/v1` API 주소로 바꾼다.

## 실제 연동 smoke test

프로젝트의 `measure-langgraph` 디렉터리에서 실행한다.

```powershell
cd measure-langgraph
```

### 검색 top_k

`internalCount`는 최종 내부 추천 개수이고 `retrievalTopK`는 Dify에서 검색할 문서
개수다. 두 값은 독립적이다. `retrievalTopK`를 생략하면 다음 규칙으로 자동 계산한다.

```text
min(10, max(5, internalCount × 3))
```

| internalCount | 자동 retrievalTopK |
|---:|---:|
| 1 | 5 |
| 2 | 6 |
| 3 | 9 |
| 4 이상 | 10 |
| 0 | 0 |

직접 지정하려면 payload에서는 `retrievalTopK`, CLI에서는 `--retrieval-top-k`를
사용한다. 입력값은 1~10 범위로 보정된다.

```powershell
python -m native.smoke_live `
  --step cause `
  --phenomenon "도어트림 상단 단차 불량" `
  --internal-count 1 `
  --external-count 1 `
  --retrieval-top-k 5
```
내부만 확인:

```powershell
python -m native.smoke_live `
  --step cause `
  --phenomenon "도어트림 상단 단차 불량" `
  --internal-count 2 `
  --external-count 0
```

외부만 확인:

```powershell
python -m native.smoke_live `
  --step cause `
  --phenomenon "도어트림 상단 단차 불량" `
  --internal-count 0 `
  --external-count 2
```

내부와 외부를 함께 확인:

```powershell
python -m native.smoke_live `
  --step cause `
  --phenomenon "도어트림 상단 단차 불량" `
  --internal-count 2 `
  --external-count 2
```

대책 추천은 원인을 함께 전달한다.

```powershell
python -m native.smoke_live `
  --step countermeasure `
  --phenomenon "도어트림 상단 단차 불량" `
  --cause "설비 정렬 편차" `
  --internal-count 2 `
  --external-count 2
```

## 전체 debug 로그

`--debug`를 지정하면 LangGraph를 `stream_mode=["debug", "values"]`로 실행해
노드 task 시작·종료, 노드 입력·출력, 단계별 전체 State를 콘솔과 파일에 기록한다.

```powershell
python -m native.smoke_live `
  --step cause `
  --phenomenon "도어트림 상단 단차 불량" `
  --internal-count 2 `
  --external-count 2 `
  --debug `
  --log-file native/logs/smoke.log
```

`--log-file`을 생략해도 기본 파일 `native/logs/smoke.log`에 append된다. 로그에는
다음 내용이 포함된다.

- LangGraph debug/values 이벤트 전체
- Dify 검색 질의·요청·응답 원문, 청크 수, 소요 시간
- 내부 OpenAI 요청·응답, 후보 수, 소요 시간
- 외부 OpenAI 요청·응답, response ID, 소요 시간
- 외부 호출 실패 시 fallback 원인과 traceback
- 최종 결과

API 키와 `Authorization` 헤더는 로그에 기록하지 않으며, debug 이벤트에 민감한
키 이름이 나타나면 값은 `***`로 치환한다. `native/logs/`는 `.gitignore`에 포함된다.
## 테스트

Fake 기반 노드·그래프 회귀 테스트:

```powershell
python -m unittest native.test_nodes -v
```

Dify/OpenAI 어댑터 요청·응답 계약 테스트:

```powershell
python -m unittest native.test_live_adapters -v
```

`test_live_adapters`는 실제 네트워크를 호출하지 않는다. 실제 인증·네트워크·모델
권한까지 확인하려면 위 `smoke_live` 명령을 사용한다.
