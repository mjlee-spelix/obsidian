---
tags: [지식, LLM, AI, JSON, 트러블슈팅]
date: 2026-04-07
---
# LLM - JSON Over-escape 버그와 복구 패턴

## 핵심
- LLM이 긴 JSON 배열을 생성하다 중간/후반 row에서 갑자기 quote를 `\"`로 over-escape 하는 알려진 버그
- 모델이 escape sequence를 *기능적 문법*이 아닌 *텍스트 패턴*으로 학습해서 발생
- gpt-oss, GPT-4o, Claude, Gemini 모두에서 보고됨. 오픈/소형 모델일수록 빈도 높음
- 해결: JSON 모드만으로는 100% 못 막음 → **fallback repair 파서가 표준 패턴**

## 상세

### 증상 — 우리가 본 실제 케이스

NL2SQL Agent가 두 번 iteration해서 두 wrapper를 출력했는데, 두 번째 wrapper의 마지막 row만 over-escape됨.

```
{
  "raw_data": [
    {"order_month":1,"monthly_revenue":"4276427.18", ...},
    {"order_month":2,"monthly_revenue":"3565878.91", ...},
    {"order_month":3, ...},
    {"order_month":4, ...},
    {"order_month":5, ...},
    {"order_month\":6,\"monthly_revenue\":\"47491.55\",\"order_count\":905,...}
  ]
}
```

마지막 row 안의 모든 `"`가 `\"`로 escape되어 있음. JSON 파서는 이걸 키 안의 escaped quote로 해석해서 row 전체가 invalid가 됨.

### 발생 원인

1. **토큰 분포 흔들림**: 긴 배열 출력 중 모델이 "지금 string literal 안에 있다"고 착각. 한 row에서 시작되면 그 row 전체가 escape mode로 잠김
2. **학습 데이터 모방**: 학습 셋에 escaped JSON 문자열(`"{\"key\":\"value\"}"`)이 자주 등장 → 출력 중 그 패턴으로 빠짐
3. **JSON-in-JSON 구조 취약성**: aider 벤치마크에서 "코드를 JSON으로 감싸달라고 하면 모델 정확도가 크게 떨어진다"고 측정됨

### 흔한 LLM JSON 버그 종류

| 버그 | 예시 |
|---|---|
| Over-escape | `{"key\":\"value\"}` |
| **Invalid escape sequence** | `{"mom\_growth\_pct":7}` (`\_`는 JSON valid escape 아님) |
| Raw control char | string 안에 escape 안 된 `\n`, `\t` (Claude on Bedrock에서 자주) |
| Truncated JSON | 닫는 `}` / `]` 누락 |
| Trailing comma | `{"a":1,}` |
| Markdown fence | ` ```json ... ``` ` 으로 감쌈 |
| Smart quote | `"key": "value"` |
| Single quote | `{'key': 'value'}` |
| Unquoted key | `{key: "value"}` |
| 설명 텍스트 prepend | `Here is the JSON: {...}` |

### Invalid Escape Sequence 상세

JSON 스펙상 valid escape는 다음 9개뿐:
`\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t`, `\uXXXX`

LLM은 종종 마크다운/RST/LaTeX 습관에 영향받아 다음과 같은 invalid escape를 출력함:
- `\_` (마크다운의 underscore escape 모방)
- `\.` (정규식/RST 모방)
- `\-`, `\+`, `\*` (마크다운 escape 모방)

이런 게 string 안에 들어가면 Python `json.loads`는 `Invalid \escape` 에러로 즉시 fail. 단순 `\" → "` 복구만으로는 못 잡음.

### 해결 방법

#### 1. Fallback Repair 파서 (가장 안정적)
1차로 strict 파싱 시도 → 실패하거나 결과가 비면 복구 후 재시도

```python
# 1차: strict raw_decode 루프
def extract_wrappers(s):
    decoder = json.JSONDecoder()
    results = []
    idx = 0
    while idx < len(s):
        pos = s.find('{', idx)
        if pos == -1:
            break
        try:
            obj, end = decoder.raw_decode(s, pos)
            if isinstance(obj, dict) and ('sql' in obj or 'raw_data' in obj):
                results.append(obj)
            idx = end
        except json.JSONDecodeError:
            idx = pos + 1
    return results

# 2차 폴백: over-escape + invalid escape 복구
if best is None or score(best) == 0:
    repaired = text.replace('\\"', '"')                       # \" → "
    # JSON valid escape가 아닌 백슬래시 제거 (\_, \., \-, \space 등)
    repaired = re.sub(r'\\(?!["\\/bfnrtu])', '', repaired)
    repaired = re.sub(r'}"(\s*[,\]])', r'}\1', repaired)      # }", → },
    repaired = re.sub(r'(\s*[,\[])"\{', r'\1{', repaired)    # ,"{ → ,{
    wrappers2 = extract_wrappers(repaired)
```

**핵심 정규식: `r'\\(?!["\\/bfnrtu])'`**
- `\\` → 입력의 백슬래시 1개 매치
- `(?!["\\/bfnrtu])` → negative lookahead. 다음 글자가 valid JSON escape char(`"`, `\`, `/`, `b`, `f`, `n`, `r`, `t`, `u`) 중 하나면 매치 안 함
- 매치되면 `''`로 치환 → 백슬래시만 제거, 다음 글자는 유지
- `\_` → `_`, `\.` → `.`, `\-` → `-`로 정규화
- `\n`, `\t`, `\\`, `\u00FF` 등 valid escape는 그대로 보존

#### 2. `json_repair` 라이브러리 (Python)
무효 JSON 자동 복구 라이브러리. 따옴표/괄호/콤마 구조 분석 후 보정. 우리 폴백 코드의 "프로용 버전"

#### 3. JSON Mode / Structured Outputs
OpenAI/Anthropic이 제공하는 강제 스키마 모드. 그래도 100%는 못 막음

#### 4. 프롬프트 강화 (보조)
```
raw_data 배열 안의 dict는 평범한 JSON으로 출력하세요.
키와 값에 \" 같은 escape를 절대 사용하지 마세요.
```
+ temperature 낮추기 (0.2 → 0.1)
→ 빈도는 줄지만 완전히 막진 못함

### 핵심 교훈

- production에서 LLM JSON 처리할 땐 **항상 repair 단계**를 두는 게 표준
- JSON 모드/structured output을 믿지 말고 안전망 둘 것
- 모델별 특성 차이: GPT-4o/Claude > 오픈 모델(gpt-oss 등) — 오픈 모델 쓸 땐 fallback 필수
- aider의 결론: **"코드/구조화 데이터는 JSON으로 감싸지 말고 따로 처리하라"**

## 관련 노트
- [[Dify Agent - LLM 카테고리명 환각 대응]]
- [[Dify Agent - null값 KeyError 처리]]
- [[Dify Agent - gpt-oss vLLM Function Calling 트러블슈팅]]
- [[gpt-oss vLLM Function Calling - 원인 분석 심층 리포트]]

## 참고 자료
- [OpenAI Community - Improperly Escaped Quotes in Returned JSON Values](https://community.openai.com/t/improperly-escaped-quotes-in-returned-json-values/323980)
- [OpenAI Community - JSON mode escaping quotation marks](https://community.openai.com/t/how-do-i-ensure-that-json-mode-properly-escapes-quotation-marks/619138)
- [aider - LLMs are bad at returning code in JSON](https://aider.chat/2024/08/14/code-in-json.html)
- [Tutorial on Using json_repair in Python](https://medium.com/@yanxingyang/tutorial-on-using-json-repair-in-python-easily-fix-invalid-json-returned-by-llm-8e43e6c01fa0)
- [Stop begging for JSON - Charlie Guo](https://www.ignorance.ai/p/stop-begging-for-json)
