---
tags: [지식, dify, AI-Agent, LLM, vLLM, function-calling, gpt-oss, 트러블슈팅]
date: 2026-03-30
---
# Dify Agent - gpt-oss vLLM Function Calling 트러블슈팅

## 핵심
- gpt-oss 모델을 Ollama → vLLM으로 전환하고, 도구를 multi-action 방식으로 변경했을 때 function calling이 정상 동작하지 않음
- 원인은 크게 3가지: chat_template 미적용, tool-call-parser 포맷 불일치, multi-action 도구 설계의 취약성
- 해결책: `--chat-template` 명시 지정 + 메타데이터 프롬프트 내장으로 도구 호출 최소화

---

## 1. 환경 및 배경

### 목표
PostgreSQL DB를 조회하는 기업 매출 분석 Agent (Dify 기반, function calling 전략)

### DB 구조
- 스키마: `agent_demo`
- 테이블 7개: `analytics_customer_rfm`, `analytics_customer_stats`, `analytics_inventory_status`, `analytics_monthly_trend`, `analytics_product_sales_rank`, `analytics_promotion_effect`, `analytics_sales_fact`

### 모델 정보 (gpt-oss)
두 모델 모두 동일한 아키텍처 (`GptOssForCausalLM`, MoE 구조)

| 항목 | gpt-oss-20b | gpt-oss-120b |
|---|---|---|
| architecture | GptOssForCausalLM | GptOssForCausalLM |
| model_type | gpt_oss | gpt_oss |
| hidden_size | 2880 | 2880 |
| layers | 24 | 36 |
| experts (total/active) | 32/4 | 128/4 |
| quantization | mxfp4 | mxfp4 |
| vocab_size | 201088 | 201088 |

---

## 2. 두 가지 Setup 비교

### Setup 1: 정상 동작
- **모델**: gpt-oss-20b (Ollama)
- **에이전트 전략**: function calling
- **도구**: SQL Executor Tool
  - 파라미터: `query` 하나
  - 모델이 직접 SQL을 생성해서 전달
- **프롬프트**: 스키마/테이블/컬럼 조회 SQL을 프롬프트에 안내, 모델이 알아서 discovery
- **동작**: round당 tool 호출 1회, 순차적으로 discovery → 분석 쿼리 실행
- **한계**: `analytics_monthly_trend` 같은 사전 집계 테이블을 무시하고 `analytics_sales_fact`만 참조

### Setup 2: 비정상 동작 (개선 시도)
- **모델**: gpt-oss-120b (vLLM)
- **에이전트 전략**: function calling
- **도구**: PostgresqlTool (Dify workflow tool)
  - 파라미터: `action`, `table_name`, `query`
  - action별로 미리 쿼리를 내장 (get_schema_list, get_table_list, get_columns, execute_sql)
  - 모델은 action만 선택하면 됨
- **의도**: 모델이 잘못된 쿼리를 보내는 문제를 방지하려 도구 안에 쿼리를 미리 넣어둠

---

## 3. Setup 2에서 발생한 문제들

### 문제 1: 테이블명 Hallucination
- `get_table_list` 결과로 `analytics_sales_fact`, `analytics_monthly_trend` 등이 정확히 반환됨
- 그런데 `get_columns` 호출 시 `"sales"`, `"orders"`, `"salesorderheader"` 등 **존재하지 않는 테이블명**을 넣음
- 이는 모델이 AdventureWorks DB의 원본 테이블명을 사전지식에서 추측한 것
- 빈 결과 `{"data": []}` 만 반환되고, "해당 테이블이 없습니다" 같은 에러 메시지가 없어서 모델이 원인 파악 불가

### 문제 2: 무한 루프
- get_schema_list → get_table_list → 잘못된 get_columns → 다시 get_schema_list → 반복
- round 1에서만 tool 호출 12회 이상, 유의미한 결과 없음
- 같은 action을 연속 3~4회 반복 호출

### 문제 3: 병렬 Function Calling
- 120b 모델이 한 번의 응답에서 **여러 tool call을 동시 발행**
- `get_table_list`와 `get_columns`를 동시에 호출
- `get_table_list` 결과를 안 기다리고 `get_columns`의 table_name을 추측 → hallucination 발생
- 반면 Setup 1의 20b는 round당 1회 순차 호출이라 이 문제 없었음

#### 병렬 호출이 발생하는 이유
- **vLLM 측**: `--enable-auto-tool-choice`가 모델에게 tool call 개수를 자유롭게 결정하게 허용 (제한 옵션 없음)
- **모델 측**: 병렬 호출 여부는 모델의 학습 특성에 따라 결정됨. 120b는 여러 개를 한꺼번에 호출하는 경향
- vLLM에 `--max-tool-calls-per-turn` 같은 옵션은 현재 없음

### 문제 4: 데이터 있는데 "데이터 없음" 출력
- 모델이 제대로 실행해서 결과를 받았음에도 최종 응답에서 "데이터가 없다"고 보고하는 경우 발생

---

## 4. 다른 조합 시도 결과

| 조합 | 결과 | 원인 |
|---|---|---|
| 20b + Ollama + SQL Executor (Setup 1) | **정상** | 단순한 tool, 호환되는 parser, 순차 호출 |
| 120b + vLLM + PostgresqlTool (Setup 2) | **실패** - 병렬호출 + hallucination + 루프 | 모델이 결과 안 기다리고 추측 |
| 20b + vLLM + PostgresqlTool | **실패** - 내부 추론을 텍스트로 출력, 1 round 종료 | tool-call-parser와 모델 출력 포맷 불일치 |
| 20b + Ollama + PostgresqlTool | **실패** - `PluginInvokeError: KeyError: ''` | Dify가 Ollama 응답의 multi-parameter 매핑 실패 |

핵심 패턴: **PostgresqlTool(multi-action 설계)로 바꾸면 어떤 조합이든 문제 발생**

---

## 5. 근본 원인 분석

### 원인 1: chat_template 미적용 (가장 중요)

gpt-oss 모델은 자체 chat_template.jinja 파일을 갖고 있으나, `tokenizer_config.json`에 `chat_template` 필드가 **정의되어 있지 않음**. 별도 jinja 파일로만 존재.

vLLM이 이 파일을 자동으로 읽지 않으면 **기본 fallback 템플릿**이 사용됨. fallback은 tool call 관련 특수 토큰(`<|call|>`, `<|channel|>`, `<|start|>`)을 전혀 처리하지 않는 단순한 포맷.

#### vLLM의 템플릿 탐색 순서
```
1. --chat-template 플래그로 명시한 jinja 파일
2. tokenizer_config.json 안의 chat_template 필드
3. 모델 디렉토리의 chat_template.jinja 파일 (버전에 따라 지원/미지원)
4. 위 전부 없으면 → 기본 fallback (role: content 단순 나열)
```

#### 확인 방법
```bash
grep -i "chat.template\|jinja" /data/vllm_8000.log | head -5
```

### 원인 2: gpt-oss의 tool call 포맷 ≠ OpenAI 포맷

chat_template.jinja 분석 결과, gpt-oss의 tool call 포맷:

```
# 모델이 tool call할 때
<|start|>assistant to=functions.{tool_name}<|channel|>commentary json<|message|>{arguments}<|call|>

# tool 결과를 돌려줄 때
<|start|>functions.{tool_name} to=assistant<|channel|>commentary<|message|>{content}<|end|>

# 최종 사용자 응답
<|start|>assistant<|channel|>final<|message|>{content}<|end|>
```

`--tool-call-parser openai`가 기대하는 표준 OpenAI JSON 형식과 **완전히 다름**.

#### gpt-oss의 채널 시스템
| 채널 | 역할 |
|---|---|
| `analysis` | 내부 추론 (Chain of Thought) |
| `commentary` | tool call 및 tool 응답 |
| `final` | 최종 사용자 응답 |

→ 20b에서 "내부 추론이 텍스트로 출력된" 문제는 vLLM이 `analysis` 채널과 `final` 채널을 분리하지 못한 것

#### chat_template의 중요한 제약
```
{#- We assume max 1 tool call per message #}
```
템플릿에 **메시지당 최대 1개의 tool call**이 명시되어 있음. 정상적으로 템플릿이 적용되면 병렬 호출 문제도 해결될 가능성 있음.

### 원인 3: Multi-action 도구 설계의 취약성

PostgresqlTool의 `get_columns` action에서 `table_name`이 **자유 텍스트 입력**.
- 모델이 아무 값이나 넣을 수 있음
- 잘못된 값 → 빈 결과 → 에러 메시지 없음 → 모델이 원인 파악 못함 → 루프
- Dify의 Ollama 연동 시 multi-parameter 매핑에서 KeyError 발생

---

## 6. 현재 vLLM 실행 명령어

```bash
CUDA_VISIBLE_DEVICES=0,1,2,3 nohup /root/miniconda3/envs/py312_sr/bin/vllm serve /install_file_backup/tessinu/gpt-oss-120b \
  --tensor-parallel-size 4 \
  --port 8000 \
  --host 0.0.0.0 \
  --gpu-memory-utilization 0.90 \
  --max-model-len 131072 \
  --enable-auto-tool-choice \
  --tool-call-parser openai \
  > vllm_8000.log 2>&1 &
```

### 문제점
- `--chat-template` 미지정 → gpt-oss 전용 chat_template.jinja가 적용 안 될 가능성
- `--tool-call-parser openai` → gpt-oss의 커스텀 포맷(`<|call|>` 토큰 기반)과 불일치

### 수정된 명령어 (시도 필요)
```bash
CUDA_VISIBLE_DEVICES=0,1,2,3 nohup /root/miniconda3/envs/py312_sr/bin/vllm serve /install_file_backup/tessinu/gpt-oss-120b \
  --tensor-parallel-size 4 \
  --port 8000 \
  --host 0.0.0.0 \
  --gpu-memory-utilization 0.90 \
  --max-model-len 131072 \
  --enable-auto-tool-choice \
  --tool-call-parser openai \
  --chat-template /install_file_backup/tessinu/gpt-oss-120b/chat_template.jinja \
  > vllm_8000.log 2>&1 &
```

→ `--tool-call-parser openai`가 gpt-oss 커스텀 포맷과 호환되는지는 추가 확인 필요. 안 되면 모델 제공자에게 권장 parser 설정 문의.

---

## 7. 해결 방안

### 방안 1: 메타데이터 프롬프트 내장 + SQL Executor Tool (가장 확실)

Setup 1의 SQL Executor Tool로 돌아가되, 프롬프트에 전체 스키마/컬럼 정보를 미리 넣기.

**변경 전 (프롬프트의 [DB Schema Discovery]):**
```
SQL Executor Tool로 아래 쿼리를 필요할 때마다 실행하세요.
- 스키마 목록: SELECT schema_name FROM ...
- 테이블 목록: SELECT table_name FROM ...
- 컬럼 정보: SELECT column_name, data_type FROM ...
```

**변경 후 (프롬프트의 [DB Schema]):**
```
아래는 agent_demo 스키마의 전체 테이블/컬럼 정보입니다.
이 정보만으로 SQL을 생성하여 SQL Executor Tool에 전달하세요.
스키마/테이블/컬럼 조회 쿼리를 별도로 실행할 필요 없습니다.

1. analytics_monthly_trend (월별 매출 트렌드, YoY/MoM)
   - order_year (integer), order_month (integer)
   - monthly_revenue (numeric), order_count (bigint), customer_count (bigint)
   - avg_margin_pct (numeric), cumulative_revenue (numeric)
   - prev_year_revenue (numeric), prev_month_revenue (numeric)
   - yoy_growth_pct (numeric), mom_growth_pct (numeric)

2. analytics_sales_fact (주문-제품 단위 매출 팩트)
   ...

[테이블 선택 가이드]
- 월별 매출/성장률 → analytics_monthly_trend (사전 집계됨, 우선 사용)
- 제품별 매출 순위 → analytics_product_sales_rank
- 고객 분석/이탈 → analytics_customer_stats 또는 analytics_customer_rfm
- 재고 현황 → analytics_inventory_status
- 프로모션 효과 → analytics_promotion_effect
- 상세 주문 데이터/커스텀 집계 → analytics_sales_fact (위 테이블로 해결 안 될 때만)
```

**[Processing] 섹션도 함께 수정:**
```
3. 위 [DB Schema]에서 적절한 테이블과 컬럼을 선택하세요.
4. SQL 생성 후, 사용한 테이블과 컬럼이 [DB Schema]에 존재하는지 반드시 검증하세요.
   [DB Schema]에 없는 테이블이나 컬럼은 절대 사용하지 마세요.
```

**효과:**
- Discovery 단계 제거 → 병렬 호출/루프 문제 원천 차단
- tool 호출 1~2회로 감소
- 테이블 선택 가이드로 사전 집계 테이블 우선 사용 (Setup 1의 한계도 해결)
- 검증된 단순 도구 사용 → 호환성 문제 없음

### 방안 2: vLLM에 chat_template 명시 지정

`--chat-template` 플래그 추가로 gpt-oss 전용 템플릿 적용. 이를 통해:
- `<|call|>` 토큰 기반 tool call 포맷이 정상 사용됨
- 1 tool call per message 제약 적용
- analysis/commentary/final 채널 분리 정상 동작

### 방안 3: get_columns에 validation 추가

도구 워크플로우에 코드 노드를 추가해서, table_name이 존재하지 않으면 명확한 에러 반환:
```
"ERROR: 'sales' 테이블은 존재하지 않습니다. 사용 가능한 테이블: analytics_customer_rfm, analytics_customer_stats, ..."
```

### 방안 4: 프롬프트 강화 (보조적)

```
[CRITICAL RULE]
- get_columns 호출 시 table_name은 반드시 get_table_list 결과에서 반환된 정확한 이름만 사용하세요.
- "sales", "orders" 등 임의의 테이블명을 절대 사용하지 마세요.
- 같은 action을 2번 이상 연속 호출하지 마세요.
- 한 번에 하나의 action만 호출하세요.
```

### 우선순위
**방안 1 (즉시 적용)** → 방안 2 (vLLM 재시작 필요) → 방안 3, 4 (보조)

---

## 8. 미확인 사항

- [ ] vLLM이 chat_template.jinja를 자동으로 읽고 있는지 로그 확인
- [ ] `--chat-template` 명시 지정 후 동작 변화 확인
- [x] `--tool-call-parser openai`가 gpt-oss 커스텀 포맷과 호환되는지 확인 → **호환됨 (조건부)**
- [ ] 20b 모델의 function calling 학습 수준 (20b가 tool calling을 덜 학습했을 가능성)
- [ ] 현재 서버의 vLLM 버전 확인 (v0.10.2 이상인지)

---

## 9. `--tool-call-parser openai` + gpt-oss 호환성 조사 결과

### 결론: 호환됨. 단, vLLM v0.10.2 이상 필요.

gpt-oss용 tool call 파서가 vLLM PR #22386으로 추가되었고, v0.10.2부터 사용 가능.
그 이전 버전에서는 tool call이 `content` 필드에 plain text로 반환되고 `tool_calls` 배열은 비어있는 문제 발생
(vLLM Issue #22337 — 20b에서 겪은 "내부 추론이 텍스트로 출력" 문제와 정확히 동일한 증상).

### 공식 권장 설정

```bash
vllm serve openai/gpt-oss-120b \
  --enable-auto-tool-choice \
  --tool-call-parser openai \
  --reasoning-parser openai_gptoss   # gpt-oss의 analysis 채널(CoT) 처리용
```

- `--tool-call-parser openai`: gpt-oss의 tool call 포맷 파싱 (v0.10.2+)
- `--reasoning-parser openai_gptoss`: gpt-oss의 chain-of-thought(analysis 채널) 분리 처리
- `--chat-template`: 공식 문서에서는 별도 지정 안 함 (올바른 버전이면 자동 처리 추정)

### 전용 빌드

gpt-oss 지원은 초기에 전용 빌드로 먼저 배포됨:
```bash
uv pip install --pre vllm==0.10.1+gptoss \
    --extra-index-url https://wheels.vllm.ai/gpt-oss/ \
    --extra-index-url https://download.pytorch.org/whl/nightly/cu128 \
    --index-strategy unsafe-best-match
```

### 버전 확인 방법

```bash
/root/miniconda3/envs/py312_sr/bin/python -c "import vllm; print(vllm.__version__)"
```

v0.10.2 미만이면 gpt-oss tool calling이 근본적으로 안 되므로, 업그레이드가 필요.

### 참고 자료
- [GPT-OSS - vLLM Recipes](https://docs.vllm.ai/projects/recipes/en/latest/OpenAI/GPT-OSS.html)
- [vLLM Issue #22308 - gpt-oss tool-call-parser 설정](https://github.com/vllm-project/vllm/issues/22308)
- [vLLM Issue #22337 - gpt-oss-120b tool calls 문제](https://github.com/vllm-project/vllm/issues/22337)
- [OpenAI - How to run gpt-oss with vLLM](https://developers.openai.com/cookbook/articles/gpt-oss/run-vllm)

---

## 10. Setup 3 테스트 결과: 메타데이터 프롬프트 내장 + SQL Executor Tool (vLLM 120b)

### 설정
- 모델: gpt-oss-120b (vLLM)
- 도구: SQL Executor Tool (파라미터: query 하나)
- 프롬프트: 전체 스키마/컬럼 정보 내장 + 테이블 선택 가이드
- 질문: "올해 월별 매출을 분석하세요"

### 개선된 점
- 테이블 선택 정확: `analytics_monthly_trend`를 올바르게 선택 (Setup 1의 한계 해결)
- SQL 문법 정확: `agent_demo.analytics_monthly_trend`, `WHERE order_year = 2025`, `ORDER BY order_month ASC`
- hallucinated 테이블명 문제 해소 (discovery 단계 없으므로)

### 여전히 남은 문제 3가지

#### 문제 1: 같은 쿼리를 2번 병렬 호출
```json
"tool_name": "sql_executor_tool;sql_executor_tool"
```
동일한 SQL을 2번 동시에 보냄. 도구가 단순해져도 **병렬 호출 습관은 여전**.
→ vLLM의 `--reasoning-parser openai_gptoss` 미설정이 원인 (채널 분리 안 됨)

#### 문제 2: Round 1에서 가짜 데이터 hallucination (가장 심각)

tool이 반환한 **실제 데이터**:
```
1월: $4,276,427.18  |  2월: $3,565,878.91  |  3월: $4,987,901.05
4월: $5,222,758.03  |  5월: $1,908,059.49  |  6월: $47,491.55
(7~12월 데이터 없음 — 'Today' 기준 6월 30일이므로 정상)
```

Round 1 모델이 출력한 **가짜 데이터**:
```
1월: $1,235,678.90  |  2월: $1,102,345.67  |  ... | 12월: $1,845,123.45
(12개월 전부 있고, 숫자가 하나도 안 맞음)
```

tool 결과를 **완전히 무시**하고 12개월치 데이터를 통째로 hallucinate.

**원인**: gpt-oss의 analysis 채널(CoT)에서 tool call 전에 예상 답변을 미리 생성하는 특성이 있음.
`--reasoning-parser openai_gptoss`가 없으면 이 CoT 출력이 실제 응답에 섞여 나옴.

#### 문제 3: 최종 출력에 JSON 2개가 연결됨
```
최종 text = Round 1 가짜 JSON + Round 2 실제 JSON (두 개가 붙어나옴)
```
- Round 1: analysis 채널의 가짜 답변 (12개월, hallucinated 숫자)
- Round 2: 실제 tool 결과 기반 답변 (1~6월 실제 데이터 + 7~12월 "데이터 없음")
- JSON 파싱 불가능한 형태로 최종 출력

### 원인 분석: `--reasoning-parser openai_gptoss` 미설정

```
gpt-oss 내부 동작:
  1. analysis 채널 → "월별 매출을 분석해야겠다" + 예상 답변 미리 생성 (hallucinate)
  2. commentary 채널 → tool call 실행 → 실제 데이터 수신
  3. final 채널 → 실제 데이터 기반 응답 생성

reasoning-parser 없을 때:
  → analysis 채널 출력이 content로 혼입
  → Round 1에 가짜 답변이 실제 output으로 취급
  → Round 2의 실제 답변과 합쳐져 JSON 2개 출력

reasoning-parser 있을 때 (기대):
  → analysis 채널 → reasoning_content로 분리 (사용자에게 안 보임)
  → final 채널만 content로 전달
  → 깔끔한 단일 JSON 출력
```

### 필요한 조치

**1. vLLM 설정 변경 (최우선)**
```bash
CUDA_VISIBLE_DEVICES=0,1,2,3 vllm serve /install_file_backup/tessinu/gpt-oss-120b \
  --tensor-parallel-size 4 \
  --port 8000 \
  --host 0.0.0.0 \
  --gpu-memory-utilization 0.90 \
  --max-model-len 131072 \
  --enable-auto-tool-choice \
  --tool-call-parser openai \
  --reasoning-parser openai_gptoss
```

**2. vLLM 버전 확인**
```bash
/root/miniconda3/envs/py312_sr/bin/python -c "import vllm; print(vllm.__version__)"
```
v0.10.2 미만이면 `--reasoning-parser openai_gptoss` 자체가 미지원 → 업그레이드 필요.

---

## 관련 노트
- [[Dify Agent - LLM 카테고리명 환각 대응]]
- [[Dify Agent - null값 KeyError 처리]]
- [[SQL - % 와일드카드 이스케이프]]
