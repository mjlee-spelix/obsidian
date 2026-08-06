---
tags:
  - langgraph
  - dify
  - AI-Agent
  - 과제5
  - 개발
created: 2026-06-30
source:
  - dify/대책서 작성 도우미 v0.0.7.yml
  - dify/과거 품질 이력 수집·저장 에이전트 v0.0.2.yml
  - dify/검색 DB 증분 색인 v0.0.2.yml
---

# 대책서 작성 도우미 — Dify → LangGraph 구현 설계

Dify 워크플로우 `대책서 작성 도우미 v0.0.7`을 LangGraph로 포팅하기 위한 설계 분석.

> **진짜 목표(맥락)**: Dify는 최종 런타임이 아니라 **노드 구조를 눈으로 확인하기 위한 청사진**이다. 최종 산출물은 **자체 제작 LangGraph 기반 에이전트 빌더**이며, 이 워크플로우는 그 빌더 위에서 재현된다. 따라서 Dify에 컴포넌트를 등록하거나 Dify를 런타임으로 쓰는 방향(Dify가 지휘자)은 **아니다**. 장기적으로는 **순수 B(네이티브)** 가 이 목표에 가장 부합한다(빌더에 올릴 재사용 블록이므로). → 빌더 관점의 컴포넌트 경계는 §8 참조.

## 0. 구현 전략 — 세 버전(A · 하이브리드 · 순수B)

> 이 문서 하나로 **서로 다른 대화에서 세 가지 LangGraph 구현을 각각 빌드**한다. 단, 셋의 목적이 다르다(아래).

세 버전은 **출력(비즈니스 로직)이 모두 동일**하다. 그래서 비교의 의미가 버전쌍마다 다르다:

| 비교쌍 | 가르쳐주는 것 | 출력 차이 |
|---|---|---|
| **A ↔ 하이브리드** | **토폴로지** — Dify 복제 그래프 vs step 통합 (§2 단순화가 뭘 없앴나) | 없음 |
| **하이브리드 ↔ B** | **코드 출처** — 검증코드 복붙 vs 전면 재작성 | 없음 |
| **Dify ↔ 나머지** | 골든 기준 (회귀 검증) | 있을 수 있음 |

→ **A는 출력/회귀 비교가 목적이 아니라, "Dify를 거의 그대로 옮긴 모습"을 보존해 토폴로지·노드수를 눈으로 비교하는 학습용 베이스라인**이다. 회귀 검증은 (Dify 골든 + 네이티브 1개)면 충분하므로 A는 제품 산출물이 아님.

### 세 버전 정의

| | A (학습용 베이스라인) | 하이브리드 | 순수 B (네이티브 재구현) |
|---|---|---|---|
| 목적 | Dify→LangGraph 1:1 매핑 학습 | 빠른 1차 이식 + 골든 대조 기준선 | 장기 유지보수 코드베이스 |
| 그래프 토폴로지 | **Dify 복제 그대로** (원인/대책 분리, 노드 ≈2배) | step 통합 (§4) | step 통합 (§4) |
| 우회 노드(HTTP·언랩·json.dumps) | **분리된 노드로 보존** | ChatOpenAI 등 네이티브로 합침 | 네이티브 (동일) |
| 더미 orphan 노드 | 보존(또는 표시) | 삭제 | 삭제 |
| Dify 코드 취급 | 코드노드 거의 복붙 | "코드=요구사항" 부분 복붙 | 명세 참고만, 전부 새로 작성 |
| 산출물 성격 | 버리는 코드(학습 후 폐기) | 제품 후보 | 제품 후보 |

> ⚠️ **A도 진짜 1:1은 불가능** — Dify 지식검색은 관리형 서비스라 retriever로 재구현 필수. A는 "완전 복사"가 아니라 "**토폴로지·우회 노드를 최대한 보존**"한 버전.

**하이브리드·순수B는 §3(State)·§4(그래프)·§5(노드 명세)가 완전히 동일**하고 노드 내부 작성 방식만 다르다. **A만 토폴로지(§4)가 다르다**(복제 그래프).

권장 진행: **A(학습) → 하이브리드(리팩터링 체감) → 여유되면 B**. 셋을 독립 빌드해 출력만 비교하는 건 효과가 적다.

## 1. Dify 워크플로우의 본질

- **무상태(stateless) 추천 생성기**: 원인(cause) / 대책(countermeasure) 후보를 추천만 함
- 세션·선택·누적·저장·횟수제한은 **프론트가 소유** (Dify는 추천 생성만)
- "더 찾기" 재요청은 `previous_response_id`(OpenAI Responses API의 직전 응답 id)로 이어붙임
- **원인 분기와 대책 분기는 완전 대칭** — 동일 구조에 쿼리/프롬프트만 다름

### 전체 흐름
```
시작 → 입력 파싱/검증 → 검증 게이트(실패=에러 종료)
     → 단계 분기(cause/countermeasure)
     → [ 내부 RAG 경로  ∥  외부 LLM 경로 ]   ← 병렬
     → 병합·정렬 → 종료
```

내부 RAG 경로: `지식검색(KR) → LLM 추출(structured) → 코드 정형화`
외부 LLM 경로: `요청바디 생성 → HTTP(OpenAI Responses API)`

## 2. Dify 노드 → LangGraph 매핑

| Dify 노드 | id | LangGraph |
|-----------|-----|-----------|
| 시작 | 2003000000001 | `START` + 입력 state |
| 코드·입력 파싱/검증 | 2003000000002 | `parse_input` 노드 |
| 검증 게이트 IF/ELSE | 2003000000003 | conditional edge (`route_after_parse`) |
| 끝·에러 | 2003000000013 | `error_end` 노드 → END |
| 단계 분기 IF/ELSE | 2003000000004 | step 파라미터로 흡수 (노드 분기 불필요) |
| 지식검색·원인/대책 | 1782704397268 / 2004000000003 | `kr_retrieve` (파라미터화) |
| LLM·내부 추출 | 2004000000001 / 2004000000004 | `llm_extract` (파라미터화) |
| 코드·내부 정형화 | 2004000000002 / 2004000000005 | `format_internal` (파라미터화) |
| 코드·외부 요청바디 | 2003000000020 / 2003000000021 | `build_external_request` |
| HTTP·외부 | 2003000000006 / 2003000000010 | `call_external_llm` |
| 코드·병합·정렬 | 2003000000007 / 2003000000011 | `merge_sort` (fan-in) |
| 끝·원인/대책 | 2003000000008 / 2003000000012 | `END` |
| 코드·내부 더미 | 2003000000005 / 2003000000009 | **삭제** (orphan, 미연결 구버전) |

### 핵심 단순화 포인트 (두 버전 공통 — 그래프 차원 결정이므로 하이브리드·순수B 모두 동일하게 적용)
1. **대칭 분기 통합**: Dify는 원인/대책을 노드까지 물리적으로 복제(20+ 노드). LangGraph는 노드 함수가 `state["step"]`을 읽어 쿼리·프롬프트만 분기 → 절반으로 축소
2. **더미 노드 2개 제거**: `코드·내부 원인/대책(더미)`는 미연결 orphan(구버전 보존용). 포팅 대상 아님
3. **단계 분기를 그래프 엣지가 아닌 파라미터로**: step 화이트리스트는 이미 parse 단계에서 검증됨

## 3. State 설계

```python
from typing import TypedDict, Literal

class RecoState(TypedDict, total=False):
    # 입력
    payload: dict
    # 파싱 결과 (parse_input가 채움)
    ok: bool
    error_code: str
    message: str
    step: Literal["cause", "countermeasure"]
    phenomenon: str
    cause: str
    exclude_list: list[str]
    exclude_block: str
    internal_count: int
    external_count: int
    model: str
    previous_response_id: str
    # 내부 RAG 경로
    kr_chunks: list[dict]          # 지식검색 결과
    internal_items: list[dict]     # 정형화된 내부 후보
    # 외부 LLM 경로
    external_request_body: str
    external_response: dict
    response_id: str               # OpenAI 응답 id (round-trip)
    # 최종
    result: dict
```

> `internal_items`와 `external_response`는 **병렬 경로가 각각 다른 키에 쓰므로** reducer 충돌 없음. 같은 키를 두 경로가 동시에 쓰면 `Annotated[list, operator.add]` 같은 reducer 필요.

## 4. 그래프 구성

```python
from langgraph.graph import StateGraph, START, END

g = StateGraph(RecoState)
g.add_node("parse_input", parse_input)
g.add_node("error_end", error_end)
g.add_node("kr_retrieve", kr_retrieve)
g.add_node("llm_extract", llm_extract)
g.add_node("format_internal", format_internal)
g.add_node("build_external_request", build_external_request)
g.add_node("call_external_llm", call_external_llm)
g.add_node("merge_sort", merge_sort)

g.add_edge(START, "parse_input")

# 검증 게이트 + 병렬 fan-out
def route_after_parse(state: RecoState):
    if not state["ok"]:
        return "error_end"
    return ["kr_retrieve", "build_external_request"]  # 리스트 반환 = 병렬

g.add_conditional_edges(
    "parse_input", route_after_parse,
    ["error_end", "kr_retrieve", "build_external_request"],
)

# 내부 RAG 경로
g.add_edge("kr_retrieve", "llm_extract")
g.add_edge("llm_extract", "format_internal")
g.add_edge("format_internal", "merge_sort")

# 외부 LLM 경로
g.add_edge("build_external_request", "call_external_llm")
g.add_edge("call_external_llm", "merge_sort")

# 병합(fan-in): format_internal·call_external_llm 둘 다 끝나야 실행
g.add_edge("merge_sort", END)
g.add_edge("error_end", END)

app = g.compile()
```

- `merge_sort`는 두 선행 노드를 가지므로 **둘 다 완료 후 1회 실행**(LangGraph superstep fan-in 기본 동작)
- 내부 RAG 경로(3노드)가 외부 경로(2노드)보다 길어도 병합이 알아서 대기

## 5. 노드별 구현 매핑

> 아래는 **공통 명세**(두 버전이 똑같이 만족해야 할 동작). 각 노드 끝에 `[복붙 대상]`(하이브리드는 Dify 코드 그대로 / 순수B는 재작성) 또는 `[둘 다 재구현]`(우회라 양쪽 모두 네이티브로 새로 작성) 표기.

### parse_input (2003000000002) `[복붙 대상]`
Dify Python 코드를 거의 그대로 이식(하이브리드) / 함수 분리해 재작성(순수B). 검증 항목:
- `model` 화이트리스트: `gpt-5-nano` / `gpt-5.5` / `gpt-5.5-pro`, 외 값 → `gpt-5-nano`
- `step` 화이트리스트: `cause` / `countermeasure`, 외 → `INVALID_STEP`
- `step==cause`인데 `phenomenon` 없음 → `MISSING_PHENOMENON`
- `step==countermeasure`인데 `cause` 없음 → `MISSING_CAUSE`
- `exclude` 정규화: trim + 빈값 제거 + 순서유지 중복제거 (**여기 한 곳에서만**)
- 개수 보정: 기본 5/5, 합 최대 10, 외부부터 축소, 둘 다 0 → `INVALID_COUNT`
- `exclude_block`: 제외 항목을 프롬프트용 텍스트 블록으로 생성

가드 실패는 예외가 아니라 `ok=false + error_code`로 정상 반환 → conditional edge가 분기.

### kr_retrieve (지식검색) `[둘 다 재구현]`
- Dify 내장 지식검색 = bge-m3-ko 임베딩, 벡터 검색, top_k=10, 단일 dataset
- LangGraph: 동일 임베딩 모델(`bge-m3-ko`)로 구성한 VectorStore retriever(FAISS/Chroma/PGVector 등)
- 쿼리: `step==cause`면 `phenomenon`, `countermeasure`면 `cause`
- ⚠️ **내부 RAG는 코난 RAGops 노드로 교체 예정** — 그 시점에 KR+추출+정형화 경로/필드 변경 예상

### llm_extract (LLM 내부 추출) `[둘 다 재구현]`
- 모델: gemma-4-26b(vllm) → LangGraph에선 동일 vllm 엔드포인트 또는 대체
- **structured output 필수**: `{items: [{text, source_doc}]}` (JSON schema strict)
- LangChain `llm.with_structured_output(schema)` 사용
- 프롬프트 핵심: 검색 결과 근거만 사용, 억지로 N개 채우지 말 것, 후보마다 다른 문서, `source_doc`에 `QIR-XXXX-XXXX` 정확히, score는 매기지 말 것(시스템이 유사도로 부여)

### format_internal (코드 정형화) `[복붙 대상]`
- LLM이 뱉은 `source_doc`의 `QIR-\d{4}-\d{3,4}` 정규식 매칭 → KR chunk와 연결
- 매칭된 chunk의 score → `score = sim*100`, 메타데이터로 `ragSource`(document_id/name, dataset_id, segment_id, similarity, excerpt) 구성
- exclude/중복 제거, internal_count 개수 컷

### build_external_request (2003000000020/021) `[둘 다 재구현]`
- OpenAI **Responses API** 바디 구성:
  - `instructions` + `input` + `store: true` + `text.format`(json_schema strict, `{items:[{text, score}]}`)
  - `model`, `previous_response_id`(있으면) 주입 — 첫 호출은 prev 생략
  - `json.dumps(ensure_ascii=False)`로 안전한 직렬화

### call_external_llm (HTTP) `[둘 다 재구현]`
- `POST https://api.openai.com/v1/responses`, `Authorization: Bearer $OPENAI_API_KEY`
- **에러 전략 = default-value** (Dify): 실패 시 빈 응답으로 폴백 → 내부 결과만으로 진행
- LangGraph: `try/except`로 감싸 실패 시 `external_response={}` 반환 (워크플로우 중단 금지)
- LangChain `ChatOpenAI(use_responses_api=True, previous_response_id=...)`로 대체 가능

### merge_sort (병합·정렬) `[복붙 대상]` (단, 외부 응답 언랩 부분은 `[둘 다 재구현]`)
- 외부 응답 언랩: `output → output_text → items` (Responses API 구조)
- 내부 + 외부 합치고 **text 기준 중복제거, score 내림차순 정렬**
- `meta`: internalAdded, externalAdded, requestedInternal/External, model, **responseId**(세션 round-trip), partial(외부 요청했으나 0건이면 true)
- 출력 계약(프론트와 합의된 형태) **그대로 유지**:
```json
{ "ok": true,
  "items": [{"text","score","source":"internal|external","ragSource"}],
  "meta": {"internalAdded","externalAdded","requestedInternal",
           "requestedExternal","model","responseId","partial"} }
```
실패 시: `{"ok": false, "errorCode", "message", "items": []}`

## 6. 구현 시 주의점 / 의사결정 필요

1. **병렬 reducer**: 내부·외부가 서로 다른 state 키를 쓰므로 기본 동작으로 충분. 단, 같은 키 동시 기록을 피할 것
2. **외부 LLM 클라이언트 선택**: 원본은 raw HTTP(Dify 모델 미등록 목적). LangGraph는 (a) `httpx` raw 호출 그대로 이식 vs (b) `ChatOpenAI` Responses API 래퍼 — `previous_response_id`/`store` 동작 일치 확인 필요
3. **내부 RAG 교체 예정**: 현재 bge-m3-ko 내장 검색 → 코난 RAGops. `kr_retrieve`/`format_internal`을 인터페이스로 추상화해두면 교체 용이
4. **무상태 유지**: LangGraph checkpointer(MemorySaver 등)를 **쓰지 말 것** — 세션은 프론트 소유. `previous_response_id`만 payload로 주고받음
5. **step 대칭 파라미터화**: 쿼리 소스(phenomenon vs cause)·프롬프트·schema name(cause_items vs cm_items)만 step으로 분기

## 7. 두 버전 빌드 가이드 (다른 대화용)

### 권장 디렉토리
```
measure-langgraph/
  common/                  # 공유: JSON schema, 테스트 payload, Dify 골든 출력
  literal/                 # A (학습용 베이스라인) — Dify 복제 토폴로지
    state.py  graph.py  nodes.py
  hybrid/                  # 하이브리드
    state.py  graph.py  nodes.py
  native/                  # 순수 B
    state.py  graph.py  nodes/  (parse.py, retrieve.py, merge.py ...)
```
- `hybrid`·`native`의 `state.py`·`graph.py`는 **사실상 동일**(§3·§4 그대로). 차이는 `nodes`에 집중.
- `literal`의 `graph.py`만 토폴로지가 다름(원인/대책 분리 + 우회 노드 보존).

### 각 대화에 줄 지시문 (그대로 복사해 사용)

**▶ A(학습용 베이스라인) 빌드 대화**
> Dify 워크플로우를 **노드·연결 구조 그대로** LangGraph로 옮긴다(학습 목적). §2 단순화를 **적용하지 말 것**:
> 원인/대책 경로를 통합하지 말고 **각각 별도 노드로 복제**, 외부호출은 `요청바디 생성 → raw HTTP(httpx) → 응답 언랩`을 **분리된 노드로** 유지, 더미 orphan 노드도 표시해 둔다.
> 단 지식검색만은 관리형이라 retriever로 재구현. 출력 계약(§5)은 동일하게 맞춘다.
> 목적은 "Dify 캔버스 ↔ LangGraph 그래프"를 1:1로 대조하는 것이므로 가독성보다 **구조 충실도** 우선.

**▶ 하이브리드 빌드 대화**
> 이 설계의 §3(State)·§4(그래프)·§5(노드 명세)를 그대로 따라 LangGraph로 구현한다.
> `[복붙 대상]` 노드(parse_input, format_internal, merge_sort 본체)는 Dify YAML 코드노드(2003000000002 / 2004000000002·005 / 2003000000007·011)의 파이썬을 거의 그대로 이식한다.
> `[둘 다 재구현]` 노드(kr_retrieve, llm_extract, build_external_request, call_external_llm, 응답 언랩)는 LangChain 네이티브로 작성한다.
> 출력 계약(§5 merge_sort)을 한 글자도 바꾸지 말 것.

**▶ 순수B 빌드 대화**
> 이 설계의 §3·§4·§5 **명세(동작·출력 계약)만** 만족시킨다. Dify 코드는 참고만 하고 **모든 노드를 처음부터 새로 작성**한다.
> 검증·개수보정·exclude 정규화·병합·정렬·meta 빌드를 각각 작은 함수로 분리하고 타입힌트를 붙인다.
> 단, 검증 규칙 수치(기본 5/5·합 max 10·외부부터 축소)와 출력 계약은 §5와 정확히 일치시킬 것.

### A 비교 포인트 (출력 아님 — 구조)
A는 회귀 검증 대상이 아니라 **구조 비교용**이다. 하이브리드와 다음을 나란히 비교:
- 노드 수 (A는 ≈2배), 엣지 수, 그래프 다이어그램
- 외부호출 표현: A=3노드 분리 vs 하이브리드=ChatOpenAI 1노드
- "step 분기를 엣지로(A) vs 파라미터로(하이브리드)" 코드 차이
→ §2 단순화가 실제로 뭘 줄였는지 체감하는 게 목적.

### 교차검증 (하이브리드·순수B — 출력 회귀)
1. **골든 기준**: Dify를 정답지로 둔다. 입력은 [[3. 프로젝트/서연이화/과제5/dify 테스트 케이스.md]] 재사용.
2. 같은 payload를 **Dify · 하이브리드 · 순수B** 세 곳에 투입. (A는 출력 동일하므로 회귀엔 불포함, 필요시 참고만)
3. 출력 비교 항목: `ok`, `error_code`, `items`(text·score·source·순서), `meta`(internalAdded/externalAdded/partial/responseId 제외 나머지).
   - LLM/RAG 비결정성 때문에 items 텍스트는 완전 일치가 아닐 수 있음 → **개수·source 분포·정렬 순서·meta 카운트**가 규칙대로인지 위주로 확인.
4. **하이브리드 = Dify와 거의 1:1** 기대(검증·병합 코드가 동일하므로). 어긋나면 이식 실수.
5. **순수B**는 로직이 동치이면 하이브리드와 동일 출력. 다르면 **재작성 중 버그** 의심.
6. 에러 케이스(INVALID_STEP / MISSING_PHENOMENON / MISSING_CAUSE / INVALID_COUNT)는 결정적이므로 **3곳 완전 일치**해야 함 → 1차 회귀 테스트로 활용.

## 8. 컴포넌트 분해 (빌더 관점)

> **관점 전환**: §2·§5는 "Dify → LangGraph 포팅" 관점이다. 이 절은 **최종 목표(LangGraph 기반 에이전트 빌더)** 관점에서, 이 워크플로우를 빌더에 올릴 **재사용 블록**으로 어떻게 끊을지 정리한다.

### 전제 — Dify 제약은 이제 없다
- Dify를 런타임으로 안 쓰므로 "Dify 코드 노드 샌드박스(import 제약 등)" 같은 제약은 무의미. 전부 내 코드다.
- 따라서 **"기술적으로 컴포넌트화 안 되는 것"은 없다.** 전부 LangGraph 서브그래프/노드로 표현 가능.
- 남는 건 순수 설계 문제: **"어떤 단위로 끊어야 빌더 블록으로 재사용하기 좋은가."**

### 컴포넌트 경계 = "자기완결적 능력" 단위
빌더에서 **노드(블록)** 는 자기완결적 능력 단위로 만들고, **제어 흐름(분기·병렬·합류)** 은 빌더 캔버스가 엣지로 그린다. (Dify가 지금 하는 역할을 빌더가 LangGraph 위에서 대신함.)

| 블록 후보 | 컴포넌트화 | 빌더에서의 의미 / 파라미터 |
|---|---|---|
| 입력 파싱·검증 (`parse_input`) | ✅ | 범용 전처리 블록. 파라미터: 화이트리스트, 기본 개수, exclude 규칙 |
| **내부 RAG (검색→추출→정형화)** | ✅✅ **1순위** | "유사 사례 검색" 서브그래프. **코난 RAGops 교체 시 이 블록만 갈아끼움** → 인터페이스로 고정 |
| 외부 LLM 추천 생성 (`build→call→unwrap`) | ✅ | "LLM으로 후보 생성" 블록. 파라미터: 모델, 프롬프트, 출력 schema |
| 병합·정렬·중복제거 (`merge_sort`) | ✅ | 범용 후처리 블록. 파라미터: 정렬키, 중복제거 기준, meta 필드 |
| 제어 흐름 (검증 게이트·병렬 fan-out·fan-in) | ⬜ **컴포넌트 아님** | **빌더 캔버스가 그리는 엣지/분기 그 자체.** 노드로 빼는 게 아니라 빌더의 오케스트레이션 기능 |

> 직전 논의에서 "제어 흐름은 Dify 몫이라 컴포넌트화 불가"라 했던 이유가 **바뀐다**: 이제는 "Dify 제약"이 아니라 **"제어 흐름은 내 빌더의 캔버스가 담당할 영역"** 이기 때문. 능력은 노드로, 흐름은 캔버스로.

### 서브그래프 경계 설계 힌트
- **내부 RAG를 하나의 서브그래프**(compiled StateGraph)로 묶어 단일 블록처럼 노출 → 코난 교체가 블록 단위 스왑이 됨. §6-3의 "인터페이스 추상화"를 여기서 실현.
- 각 블록은 **입력/출력 계약을 명시한 얇은 인터페이스**로 감싸 빌더가 파라미터 UI를 자동 생성할 수 있게 한다(출력 계약은 §5 유지).
- **A(학습용 베이스라인)** 가 여기서 실질적 가치: "Dify 캔버스 노드 단위 ↔ LangGraph 그래프"를 1:1로 익혀두면, 빌더에서 **노드를 어떤 입도(granularity)로 끊을지** 감각이 그대로 이어진다.

---

# 5-1 · 증분 색인 (다른 에이전트 — 같은 빌더/코드베이스)

> §0~§8은 **5-2 대책서 작성 도우미(추천)** 포팅 설계다. 아래 §9~§11은 **5-1 수집·저장**과 **검색 DB 증분 색인**을 LangGraph로 구현하는 설계. 출처 Dify: [[3. 프로젝트/서연이화/과제5/dify/과거 품질 이력 수집·저장 에이전트 v0.0.2.yml]], [[3. 프로젝트/서연이화/과제5/dify/검색 DB 증분 색인 v0.0.2.yml]].
> 이 둘은 5-2처럼 "Dify 복제(A) vs 네이티브(B)"를 나눌 실익이 적다(우회 노드·대칭 분기가 거의 없음) → **네이티브 한 버전**으로 바로 간다.

## 9. 에이전트 5-1 — 과거 품질 이력 수집·저장 (A1~A5)

### 본질
- 비정형 개선대책서(약 7,000건)를 **문서 1건 단위**로: OCR → 6개 항목 구조화 → 정규화 → S-CLM 전송.
- **색인(A6/A7)은 5-1이 직접 안 함.** S-CLM이 DB 반영(S3)한 뒤 §10 색인 그래프를 트리거(무상태 분리·재사용).
- 6개 항목: 발생형태·귀책사유·발생장소·중요도·발생년도·대책방안.
- Dify에선 LLM 노드가 항상 실행됐지만, LangGraph는 **OCR이 6항목을 이미 주면 LLM을 건너뜀**(동적 분기).

### 전체 흐름
```
시작(트리거) → 파싱/검증 → 검증게이트(실패=에러 종료)
     → OCR(호출+JSON파싱) ─[6항목 다 있음]→ 정규화
                          └[아니면]→ LLM 구조화 추출 → 정규화
     → S-CLM 전송 → 종료
배치 7000건: 문서 리스트를 Send로 팬아웃, 문서 1건 = 위 서브그래프 1회
```

### Dify 노드(v0.0.2) → LangGraph 매핑
| Dify 노드 | LangGraph |
|---|---|
| 시작(payload) | `START` + `DocState` |
| 코드·입력 파싱/검증 | `validate` (`Command`로 분기) |
| 검증 게이트 IF/ELSE | `validate` 내부 `Command(goto)` — **게이트 노드 소멸** |
| HTTP·OCR + 코드·OCR 응답 파싱 | `ocr` (호출+파싱 1노드, `Command`로 LLM skip 결정) |
| LLM·구조화 추출 | `extract` (`with_structured_output`) |
| 코드·전처리 정규화 | `normalize` |
| HTTP·S-CLM 전송 | `send_sclm` |
| 코드·전송 결과 / 끝·완료 | 반환 + `END` |
| 끝·에러 | `validate`의 `goto=END`(error) |

### State · 그래프
```python
import httpx
from typing import TypedDict, Literal
from pydantic import BaseModel
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, RetryPolicy
from langchain_openai import ChatOpenAI

FIELDS = ["발생형태", "귀책사유", "발생장소", "중요도", "발생년도", "대책방안"]
IO_RETRY = RetryPolicy(max_attempts=3, retry_on=(httpx.HTTPError,))  # 버전따라 retry=/retry_policy=

class Fields(BaseModel):
    발생형태: str = ""; 귀책사유: str = ""; 발생장소: str = ""
    중요도: str = ""; 발생년도: str = ""; 대책방안: str = ""

extractor = ChatOpenAI(model="gemma-4-26b", base_url="http://vllm.internal/v1",
                       api_key="EMPTY").with_structured_output(Fields)

class DocState(TypedDict, total=False):
    doc_url: str; qir_id: str; metadata: dict
    ocr_text: str; prefilled: dict; extracted: dict
    record: dict; result: dict

def validate(state: DocState) -> Command[Literal["ocr", "__end__"]]:
    if not state.get("doc_url"):
        return Command(goto=END, update={"result": {"ok": False, "error": "MISSING_DOC_URL"}})
    if not state.get("qir_id"):
        return Command(goto=END, update={"result": {"ok": False, "error": "MISSING_QIR_ID"}})
    return Command(goto="ocr")

def ocr(state: DocState) -> Command[Literal["extract", "normalize"]]:   # A1~A3 + 파싱
    obj = httpx.post("http://<ocr-host>/v1/ocr",
                     json={"fileUrl": state["doc_url"], "lang": "ko"}, timeout=60).json()
    text = obj.get("text") or obj.get("fullText") or ""
    pre = {f: obj[f].strip() for f in FIELDS if isinstance(obj.get(f), str) and obj[f].strip()}
    goto = "normalize" if len(pre) == len(FIELDS) else "extract"        # 6항목 다 있으면 LLM skip
    return Command(goto=goto, update={"ocr_text": text, "prefilled": pre})

def extract(state: DocState) -> dict:                                   # A2/A4 (LLM)
    return {"extracted": extractor.invoke(state["ocr_text"]).model_dump()}

def normalize(state: DocState) -> dict:                                 # A4
    pre, ex = state["prefilled"], state.get("extracted", {})
    record = {"qirId": state["qir_id"], "sourceDoc": state["doc_url"], **state.get("metadata", {})}
    for f in FIELDS:
        record[f] = (pre.get(f) or ex.get(f) or "").strip()            # prefilled 우선, LLM 보조
    idx = f'[품질문제ID] {state["qir_id"]}\n' + "\n".join(f"[{f}] {record[f]}" for f in FIELDS)
    return {"record": record, "result": {"index_text": idx}}

def send_sclm(state: DocState) -> dict:                                 # A5
    r = httpx.post("http://<sclm-host>/api/quality-history/extract",
                   json=state["record"], timeout=30).json()
    return {"result": {"ok": True, "qirId": state["qir_id"], "sent": True,
                       "record": state["record"], "sclm": r}}

b = StateGraph(DocState)
b.add_node("validate", validate)
b.add_node("ocr", ocr, retry_policy=IO_RETRY)
b.add_node("extract", extract, retry_policy=IO_RETRY)
b.add_node("normalize", normalize)
b.add_node("send_sclm", send_sclm, retry_policy=IO_RETRY)
b.add_edge(START, "validate")
b.add_edge("extract", "normalize")
b.add_edge("normalize", "send_sclm")
b.add_edge("send_sclm", END)          # validate/ocr는 Command 분기라 edge 불필요
collect_one = b.compile()
```

### 노드 명세
- **validate**: `docUrl`/`qirId` 필수. 실패 시 `ok=false`로 `goto=END`(예외 던지지 않음 — Dify와 동일 계약).
- **ocr** `[재구현]`: OCR 호출 + JSON 파싱(`text`/`fullText`/`pages` 흔한 키), `prefilled` 6항목 수집. **6개 다 있으면 `extract`를 건너뛰고 바로 `normalize`**(Dify엔 없던 동적 스킵).
- **extract** `[재구현]`: `with_structured_output(Fields)` — OCR이 6항목을 다 주지 않은 경우만 실행.
- **normalize** `[복붙 대상]`: `prefilled`(OCR 구조화) 우선, 없으면 LLM 값. `record` 6항목 + `index_text` 구성.
- **send_sclm** `[재구현]`: `httpx` POST + `RetryPolicy`.

### 배치 7000건 — `Send` 팬아웃
```python
import operator
from typing import Annotated
from langgraph.types import Send

class BatchState(TypedDict):
    docs: list[dict]
    results: Annotated[list, operator.add]        # reducer로 자동 누적

def fan_out(state: BatchState):
    return [Send("process", {"doc": d}) for d in state["docs"]]

def process(state: dict) -> dict:
    out = collect_one.invoke({"doc_url": state["doc"]["docUrl"],
                              "qir_id": state["doc"]["qirId"],
                              "metadata": state["doc"].get("metadata", {})})
    return {"results": [out["result"]]}

bb = StateGraph(BatchState)
bb.add_node("process", process)
bb.add_conditional_edges(START, fan_out, ["process"])
bb.add_edge("process", END)
collect_batch = bb.compile()           # 실행 시 config={"max_concurrency": N}로 동시성 제한
```

## 10. 검색 DB 증분 색인 (범용 · docType 분기)

### 본질
- **색인 메커니즘은 동일, 색인 텍스트 구성만 `docType`으로 분기** → 5-1(A6/A7)과 5-2(A7)을 **하나의 그래프로 통합**(재사용).
- 두 트리거가 같은 그래프 호출: **5-1**(S3 후, `docType=quality_history` + `record` 6항목), **5-2**(작성완료 후, `docType=confirmed_countermeasure` + `phenomenon`/`cause`/`measures`).
- Dify는 `create-by-text` HTTP였지만 LangGraph는 **vector store `add_texts`로 진짜 임베딩+upsert**. **고정 id로 멱등**(재실행/재전송해도 중복 없이 갱신).

### 흐름
```
시작(트리거) → build(docType 분기 + 색인문 구성/검증)
     ─[ok]→ index(임베딩 + upsert) → 종료
     └[검증 실패]→ 에러 종료
```

### State · 그래프
```python
from langchain_openai import OpenAIEmbeddings
from langchain_postgres import PGVector      # 코난 RAGops로 교체될 자리

emb = OpenAIEmbeddings(model="bge-m3-ko", base_url="http://vllm.internal/v1", api_key="EMPTY")
store = PGVector(embeddings=emb, collection_name="quality_search_db",
                 connection="postgresql+psycopg://...")

class IdxState(TypedDict, total=False):
    payload: dict
    name: str; index_text: str; metadata: dict; uid: str; doc_type: str
    result: dict

def build(state: IdxState) -> Command[Literal["index", "__end__"]]:
    d = state["payload"]; qir = str(d.get("qirId") or "").strip()
    dt = str(d.get("docType") or "").strip()
    if not qir:
        return Command(goto=END, update={"result": {"ok": False, "error": "MISSING_QIR_ID"}})
    meta = {"qirId": qir, "docType": dt, **(d.get("metadata") or {})}

    if dt == "quality_history":                                   # 5-1
        rec = d.get("record") or d
        vals = {f: str(rec.get(f) or "").strip() for f in FIELDS}
        if not any(vals.values()):
            return Command(goto=END, update={"result": {"ok": False, "error": "EMPTY_RECORD"}})
        text = f"[품질문제ID] {qir}\n" + "\n".join(f"[{f}] {vals[f]}" for f in FIELDS)
        name = f"{qir} (과거품질이력)"
    elif dt == "confirmed_countermeasure":                        # 5-2
        cause = str(d.get("cause") or "").strip()
        raw = d.get("measures") or []
        ms = [raw] if isinstance(raw, str) else \
             [str(m.get("text") if isinstance(m, dict) else m).strip() for m in raw]
        ms = [m for m in ms if m]
        if not cause or not ms:
            return Command(goto=END, update={"result": {"ok": False, "error": "MISSING_CAUSE_OR_MEASURES"}})
        ph = str(d.get("phenomenon") or "").strip()
        text = f"[품질문제ID] {qir}\n" + (f"[현상] {ph}\n" if ph else "") + \
               f"[원인] {cause}\n[대책]\n" + "\n".join(f"- {m}" for m in ms)
        name = f"{qir} (확정대책서)"
    else:
        return Command(goto=END, update={"result": {"ok": False, "error": "INVALID_DOCTYPE"}})

    return Command(goto="index",
                   update={"name": name, "index_text": text, "metadata": meta,
                           "uid": f"{qir}:{dt}", "doc_type": dt})

def index(state: IdxState) -> dict:
    ids = store.add_texts([state["index_text"]], metadatas=[state["metadata"]],
                          ids=[state["uid"]])        # 동일 uid 재삽입 = 갱신 = 멱등 증분
    return {"result": {"ok": True, "docType": state["doc_type"], "documentId": ids[0]}}

gi = StateGraph(IdxState)
gi.add_node("build", build)
gi.add_node("index", index, retry_policy=IO_RETRY)
gi.add_edge(START, "build")
gi.add_edge("index", END)
index_agent = gi.compile()
```

### 노드 명세
- **build**: `docType` 분기로 색인문 구성. `quality_history`→6항목, `confirmed_countermeasure`→현상-원인-대책. 검증 실패 시 `Command(goto=END)`.
- **index** `[재구현]`: `store.add_texts(ids=[f"{qir}:{docType}"])` — **고정 id가 멱등의 핵심**. Dify/mock은 매 호출 uuid라 중복 row가 쌓였음(테스트에서 확인) → 여기서 개선.

### 빌더 관점 (§8 연결)
- 이 색인 그래프가 §8에서 말한 **"자기완결 재사용 블록"의 대표 사례** — 5-1·5-2가 공유하는 단일 컴포넌트.
- `store`를 인터페이스로 추상화 → **코난 RAGops 스왑**이 블록 교체 한 번으로 끝남(§6-3·§8과 동일 원칙).

## 11. Dify 테스트에서 배운 것 → LangGraph 반영

로컬 mock(OCR/S-CLM/색인) 연동 테스트에서 나온 것들이 네이티브 설계를 **더 낫게** 만든다.

1. **네트워크 중계가 사라짐**: Dify는 도커 안이라 `host.docker.internal`→LAN IP→방화벽까지 헤맴. LangGraph는 그냥 파이썬 프로세스라 `httpx`로 직접 호출(내 코드면 함수 호출로도 가능).
2. **실패를 삼키지 않음(제일 중요)**: Dify `error_strategy: default-value`가 연결 실패를 `{}`로 바꿔 진행 → 빈 record인데 `ok:true`가 나오는 착시가 있었음. LangGraph는 `RetryPolicy` 후 실패 시 명시적 에러 라우팅.
3. **색인이 진짜 벡터 upsert**: mock은 SQLite 적재였지만 `store.add_texts`는 실제 임베딩+색인. 로컬 테스트도 Chroma 임베디드면 도커 없이 됨.
4. **멱등**: 고정 id로 재전송해도 중복 없음(mock은 uuid라 매번 중복).
5. **LLM 진짜 스킵**: OCR이 6항목을 주면 `Command(goto="normalize")`로 LLM 자체를 건너뜀 → 그 경로는 vllm 없이도 동작.

### 서빙(트리거) — Dify 시작 노드 대응
```python
from fastapi import FastAPI
app = FastAPI()

@app.post("/agents/collect")   # 5-1: 단건 또는 배치(docs[])
def collect(payload: dict):
    if "docs" in payload:
        return collect_batch.invoke(payload, config={"max_concurrency": 8})
    return collect_one.invoke(payload)

@app.post("/agents/index")     # 증분 색인: 5-1(S3 후)·5-2(작성완료) 공용, docType 분기
def index(payload: dict):
    return index_agent.invoke({"payload": payload})["result"]
```
> 무상태 원칙은 5-2와 동일 — checkpointer는 옵션. 재개/멱등이 필요하면 `PostgresSaver` + `thread_id=qir_id`.

## 관련 노트
- [[3. 프로젝트/서연이화/과제5/대책서 작성 비즈니스 로직.md]]
- [[3. 프로젝트/서연이화/과제5/과제5 개요·에이전트 설계 (통합).md]]
- [[3. 프로젝트/서연이화/과제5/measure-Dify 비교 및 Dify 구현 범위.md]]
- [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 버전 기록.md]]
