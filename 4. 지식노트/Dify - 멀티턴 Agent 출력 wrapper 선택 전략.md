---
tags: [지식, dify, LLM, NL2SQL, 트러블슈팅, 패턴]
date: 2026-04-07
---
# Dify - 멀티턴 Agent 출력 wrapper 선택 전략

## 핵심
- Function calling agent가 도구를 여러 번 호출하면 출력 텍스트(`agent_text`)에 **JSON wrapper가 여러 개** 섞여 나옴 (각 iteration의 사고 과정 + 최종 답)
- 다운스트림 코드 노드에서 그중 "**진짜 최종 답**"을 골라야 함
- 흔한 휴리스틱인 `max(wrappers, key=row_count)`는 **탐색 쿼리(scheme/distinct/sample)가 진짜 답보다 row 많을 때** 깨짐
- **올바른 전략**: `pick_best = reversed(wrappers) 중 첫 번째 valid` — agent의 최종 답은 항상 가장 마지막에 emit한 wrapper

## 상세

### 문제 상황
NL2SQL Agent의 한 turn에서 `agent_text`가 다음처럼 나옴:

```json
{"sql": "SELECT DISTINCT category_name FROM ...", "raw_data": [...4 rows], "row_count": 4}
{"sql": "SELECT SUM(...) WHERE category='Bikes'", "raw_data": [{"total_bike_revenue": 2543120.75}], "row_count": 1}
{"sql": "SELECT SUM(...) WHERE category='Bikes'", "raw_data": [{"total_bbike_revenue": "17406436.39"}], "row_count": 1}
```

해석:
- **#1**: agent가 사전 탐색용으로 카테고리 목록 조회 (4 rows, 실제 도구 호출 결과)
- **#2**: 도구 호출 직전에 LLM이 환각으로 만들어낸 preview (1 row, 가짜 숫자 `2543120.75`)
- **#3**: 진짜 도구 응답 받은 후 emit한 최종 답 (1 row, 실제 숫자 `17406436.39`. 단 컬럼명에 typo `total_bbike_revenue`)

사용자에게 보여줘야 할 답은 **#3**.

### 깨지는 휴리스틱: max row_count
초기에 코드 노드는 다음처럼 작성됨:
```python
def score(o):
    return o.get('row_count', 0)

best = max(wrappers, key=score)
```

가정: "환각 preview는 짧고, 진짜 답은 데이터가 풍부하다"

이 가정의 한계:
1. 멀티턴 agent가 사전 탐색을 하면 탐색 쿼리(`DISTINCT`, `SELECT *`, `SELECT DISTINCT category`)가 row 많음
2. 진짜 답이 집계(aggregate) 결과면 1 row인 경우가 압도적으로 많음
3. → 탐색 쿼리가 항상 점수에서 이김 → 진짜 답을 못 고름

또한 Python `max`는 동치 시 **첫 번째**를 반환 → "동치면 마지막 우선"이라는 의도 자체도 표현 안 됨.

### 올바른 전략: last valid wrapper
Agent의 동작 순서를 생각해보자:
1. (선택) 사전 탐색 도구 호출 → 결과 JSON emit
2. 분석 도구 호출 → 결과 JSON emit
3. 추가 분석 → 결과 JSON emit
4. ...
5. **마지막에 최종 답 JSON emit** ← 이게 우리가 원하는 것

= **agent의 최종 답은 항상 마지막 emit**.

따라서 다음이 정답:
```python
def is_valid(o):
    if not o.get('sql'):
        return False
    rd = o.get('raw_data', [])
    if isinstance(rd, str):
        try: rd = json.loads(rd)
        except Exception: return False
    return isinstance(rd, list) and len(rd) > 0

def pick_best(ws):
    # 뒤에서부터 valid한 첫 번째 = 가장 마지막 valid wrapper
    for w in reversed(ws):
        if is_valid(w):
            return w
    return ws[-1] if ws else None
```

`is_valid`는 안전장치: 마지막 wrapper가 환각으로 깨진 경우를 대비해 한 단계 앞으로 fallback. 정상 케이스에서는 마지막 wrapper만 선택됨.

### 왜 단일턴에서는 max가 동작했나
초기 NL2SQL은 도구 1회만 호출하는 패턴이었음. 그때 wrapper는 보통 2개:
1. 도구 호출 전 환각 preview (가짜 숫자, 1 row)
2. 도구 응답 받은 진짜 답 (실제 숫자, 1~N rows)

이 경우 #2가 #1보다 row 많거나(데이터 분석 결과) 동률이면 max가 #1을 잘못 골랐을 텐데, 마침 분석 결과가 항상 더 풍부했어서 운 좋게 동작.

→ 멀티턴(사전 탐색 추가)으로 전환되는 순간 깨짐.

### 코드 노드 외에도 적용
이 패턴은 NL2SQL뿐 아니라 **모든 멀티턴 function calling agent의 출력 파싱**에 적용 가능:

| 출력 형태 | 선택 전략 |
|---|---|
| 단일 JSON wrapper | 그냥 파싱 |
| 멀티 JSON wrapper (멀티턴 agent) | **last valid** |
| 도구 응답 + 최종 답 분리 | 도구 응답을 다운스트림 코드에서 직접 사용 |

### 한계 — 컬럼명 환각은 별도 문제
이 전략은 "어떤 wrapper를 고를지"만 해결. wrapper 내부의 환각(예: `total_bike_revenue` → `total_bbike_revenue` 같은 키 typo)은 별개.

근본 대응: `raw_data` 자체를 LLM이 생성하지 말고, 코드 노드에서 **tool_response를 직접 파싱**해서 채움. NL2SQL Agent는 SQL 생성/실행만 담당, 결과 데이터는 코드가 책임.

단, Dify Agent strategy에서 tool_response를 후속 노드로 빼낼 수 있는지는 별도 확인 필요.

## 핵심 교훈
- **멀티턴 agent의 출력 wrapper 선택은 "last valid"가 정답**. 길이/카운트 기반 휴리스틱은 사전 탐색 패턴 도입 즉시 깨짐
- 휴리스틱을 만들 때는 그 휴리스틱이 어떤 가정에 의존하는지 명시할 것. 가정이 깨지면 휴리스틱도 깨짐
- Function calling agent는 거의 항상 "마지막 emit이 최종 답"이라는 강한 invariant를 가짐. 이걸 활용하는 게 안전
- LLM이 도구 호출 **직전**에 환각으로 결과 preview를 미리 출력하는 패턴이 있음 — 이건 별도 노트로 정리할 가치 있음 ([[LLM - JSON Over-escape 버그와 복구 패턴]] 참조)

## 관련 노트
- [[LLM - JSON Over-escape 버그와 복구 패턴]]
- [[Dify Agent - gpt-oss vLLM Function Calling 트러블슈팅]]
- [[Dify Agent - LLM 카테고리명 환각 대응]]
- [[Dify - DSL YAML 작성 및 관리]]
