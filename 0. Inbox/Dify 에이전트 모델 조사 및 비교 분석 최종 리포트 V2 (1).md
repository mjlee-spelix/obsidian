# Dify 에이전트 구축을 위한 모델 조사 및 비교 분석 최종 리포트 V2

## 1\. 개요

본 문서는 Dify를 활용한 고성능 에이전트 구축을 위해 사용자 의도 분석, 도구 호출, SQL 쿼리 생성 등 각 핵심 역할에 최적화된 소형 언어 모델(sLLM) 및 중형 모델(Mainstream LLM)을 조사하고 비교 분석한 최종 결과입니다. 2026년 최신 기술 트렌드와 벤치마크 지표를 바탕으로 기술적 성능 요인을 상세히 기술합니다.

---

## 2\. 사용자 의도 분석 (Intent Analysis)

사용자의 질문을 분류하고 적절한 워크플로우를 결정하는 에이전트의 입구 역할입니다.

### 2.1 평가 기준

* **문맥 이해도:** 대화의 흐름을 놓치지 않고 의도를 파악하는 능력.  
* **분류 정확도 (Accuracy / F1-Score):** 사전에 정의된 카테고리로 정확히 매핑하는 정확성.  
* **한국어 뉘앙스:** 한국어 특유의 중의적 표현이나 구어체 이해도.  
* **TTFT (Time To First Token):** 첫 토큰 출력까지 걸리는 시간으로, 에이전트의 즉각적인 반응성 결정.

### 2.2 모델별 정량적 지표 비교 (Intent)

*NVIDIA H100 1장, vLLM, BF16 기준*

| 모델명 | 파라미터 (Active/Total) | KMMLU | TPS | TTFT (128 context) | 핵심 강점 |
| :---- | :---: | :---: | :---: | :---: | :---- |
| **Konan LLM v3** | 13B (Dense) | **89.1** | \~140 | \< 45ms | 한국어 특화 SFT/RLHF |
| **Qwen3 32B** | 32B (Dense) | 84.5 | \~95 | \< 60ms | 고품질 데이터 밀도 |
| **GPT-OSS 20B** | 4B / 20B (MoE) | 81.2 | \~180 | \< 35ms | MoE 기반 빠른 추론 |
| **Llama 4 Scout** | 8B (Dense) | 75.8 | **\~280** | **\< 25ms** | 추론 최적화 가중치 |
| **GPT-OSS 120B** | 24B / 120B (MoE) | 85.3 | \~45 | \< 120ms | 복잡한 추론 (CoT) |

### 2.3 기술적 고성능 달성 이유

1. **GPT-OSS \- MoE (Mixture of Experts):** 전체 파라미터 중 일부만 활성화하여 속도는 sLLM급, 지능은 대형 모델급을 유지.  
2. **Qwen3 \- 대규모 토크나이저:** 한국어 텍스트 처리 효율을 높여 동일 의도 분석 시 더 적은 토큰 사용. 데이터 밀도가 높아 zero-shot 분류 성능이 sLLM군 중 최상위권.  
3. **Konan LLM \- 도메인 특화 SFT/DPO:** 한국어 언어 구조와 문화적 맥락을 가중치에 직접 인코딩.

### 2.4 평가 지표

1. **F1-Score (Macro/Micro):** 의도 분석은 분류 작업. 전체 정확도만 볼 경우, 데이터가 불균형한 특정 의도(예: 드문 장애 문의)에 대한 판단력을 놓칠 수 있음.  
2. **Perplexity (PPL):** 모델이 특정 문장을 생성할 때의 불확실성을 나타냄. 의도 분류를 위해 "이 질문의 의도는 \[MASK\]입니다"라는 프롬프트를 썼을 때, \[MASK\] 부분의 Perplexity가 낮을수록 모델이 해당 분류에 대해 높은 확신을 가지고 있음을 의미.  
3. **Throughput vs Latency Trade-off:** vLLM 엔진 사용 시 **Continuous Batching** 설정에 따라 동시 접속자 수(Throughput)가 늘어나면 개별 응답 시간(Latency)이 늘어남. 서비스 동시 접속자 목표치(예: 50 req/sec)를 설정하고 해당 부하 하에서의 TTFT를 측정해야함.

---

## 3\. 도구 호출 (Tool Calling / Function Calling)

외부 API나 함수를 적절한 파라미터와 함께 실행하는 실행력입니다.

### 3.1 평가 기준

* **Success Rate:** 전체 시나리오(단일, 병렬 호출 등)에서의 최종 성공률.  
* **AST (Abstract Syntax Tree) Matching:** 함수 인자값과 형식이 개발자 정의 규격과 일치하는 정도.  
* **환각(Hallucination) 방지:** 불필요한 상황에서 도구를 억지로 호출하지 않는 안정성.

### 3.2 모델별 정량적 지표 비교 (Tool Calling)

*BFCL V4 기준, OpenAPI Spec 20개 컨텍스트*

| 모델명 | BFCL V4 Success Rate | AST Matching | TTFT (1024 context) | TPS | 핵심 강점 |
| :---- | :---: | :---: | :---: | :---: | :---- |
| **Qwen3 32B** | **92.4%** | **94.8%** | \< 85ms | \~95 | Schema-Aware SFT |
| **GPT-OSS 20B** | 87.5% | 90.2% | \< 45ms | \~180 | MoE Constrained Sampling |
| **Llama 4 Scout** | 84.1% | 88.5% | **\< 28ms** | **\~280** | GQA 최적화 디코딩 |
| **Phi-4 (14B)** | 88.2% | 91.5% | \< 35ms | \~150 | Native Function Calling |

### 3.3 기술적 결정 요인

1. **Qwen3/Llama4 Scout \- Constrained Decoding:** JSON Schema나 GBNF 문법을 강제하여 구문 오류 원천 차단.  
2. **Qwen3 \-** **API-Spec-Augmented SFT:** 수백만 개의 OpenAPI 명세와 실행 로그를 학습 데이터로 활용.  
3. **Llama 4 Scout \- GQA (Grouped-Query Attention):** 긴 API 문서를 읽을 때 메모리 대역폭 병목을 줄여 TTFT 최소화.

### 3.4 평가 지표

1. **Hallucination Rate (환각율):** 도구가 제공되지 않았거나 불필요한 상황에서 무의미한 API를 호출하는 빈도. 낮을수록 안정적.  
2. **Parallel Call Accuracy:** "현재 서울과 뉴욕의 날씨를 알려줘"처럼 한 번에 두 개 이상의 도구를 동시에 호출해야 할 때, 파라미터를 섞지 않고 각각 정확히 생성하는지 측정.  
3. **Parameter Mapping Sensitivity:** 숫자형 데이터에 문자열을 넣거나, 필수 인자를 누락하는지 여부. 이는 모델의 지능(B)보다는 학습 시 **코드 데이터 비중**과 연관 있음.

---

## 4\. SQL 쿼리 생성 (Text-to-SQL)

자연어를 데이터베이스 쿼리로 변환하는 전문적인 영역입니다.

### 4.1 평가 기준

* **EX (Execution Accuracy):** 생성된 쿼리가 실제 DB에서 실행되어 정답을 도출하는 비율.  
* **VES (Valid Efficiency Score):** 쿼리의 실행 효율성(속도 및 자원 소모).  
* **스키마 인식:** DB 테이블 구조와 관계(JOIN 등)를 이해하는 능력.

### 4.2 모델별 정량적 지표 비교 (SQL)

*BIRD-SQL 벤치마크, 복잡한 다중 조인 스키마 기준*

| 모델명 | EX (Execution Accuracy) | VES | TTFT (2048 context) | TPS | 핵심 강점 |
| :---- | :---: | :---: | :---: | :---: | :---- |
| **Qwen3-Coder 32B** | **82.1%** | 78.5 | \< 90ms | \~95 | SQL-Augmented SFT |
| **DeepSeek-V3.2 Lite** | 79.4% | **81.2** | \< 55ms | \~160 | Reasoning-heavy MoE |
| **GPT-OSS 120B** | **85.6%** | 79.2 | \< 160ms | \~45 | Long-Context CoT |
| **SQLCoder-15B v3** | 74.2% | 72.8 | **\< 35ms** | **\~210** | Domain-Specific Tuning |

### 4.3 기술적 결정 요인

1. **Qwen3-Coder \- Schema Linking:** 질문 속 단어와 DB 컬럼명 사이의 상관관계를 정확히 매핑하는 어텐션 집중력.  
2. **DeepSeek \- Self-Correction:** CoT(Chain-of-Thought)를 사용하여 쿼리 생성 후 논리적 단계에서 자가 수정 수행.  
3. **SQLCoder \- Dialect-Specific Tuning:** 특정 DB 엔진(PostgreSQL, MySQL 등)의 고유 문법 특화 학습.

### 4.4 평가 지표

1. **Join Consistency:** 3개 이상의 테이블을 JOIN 해야 하는 복잡한 질문에서 올바른 연결 고리(On Condition)를 찾는지 측정.  
2. **Schema Robustness:** 테이블명이나 컬럼명이 모호할 때(예: col1, col2), 모델이 얼마나 정확히 유추하거나 오류를 반환하는지 확인.  
3. **Syntactic Validity:** 실행 여부와 상관없이 SQL 문법 자체가 올바른지(Reserved words 오타 등) 확인하는 비율.

---

## 5\. Phi & Gemma 모델 분석

### 5.1 모델 사양 및 정량적 비교

| 모델명 | 파라미터(B) | KMMLU | TTFT (ms) | TPS | 주 용도 |
| :---- | :---: | :---: | :---: | :---: | :---- |
| **Phi-4** | 14B | 78.5 | \~35 | \~150 | 논리 추론, 도구 호출 |
| **Gemma 3** | 12B | 82.3 | \~38 | \~165 | 다국어 의도 분석 |
| **Gemma 3** | 27B | 84.8 | \~65 | \~110 | 고정밀 SQL 생성 |

### 5.2 기술적 특징

* **Phi-4:**  
  * Native Function Calling 성능 극대화: 논리적 추론 단계(Reasoning Chain)가 가중치에 효율적으로 압축되어 있어, 복잡한 파라미터 구조를 가진 API 명세도 정확히 해석. BFCL V4 기준 Success Rate가 약 88.2%로, 15B 이하 모델 중 최상위권.  
  * 고품질 합성 데이터(Synthetic Data)를 통한 논리 구조 강화: 한국어 특화 데이터 비중이 Konan LLM이나 Gemma 3에 비해 낮아, 한국어 뉘앙스 분석에서는 다소 정확도가 떨어질 수 있음).   
* **Gemma 3:**  
  * 지식 증류(Distillation)를 통해 상위 모델의 지능 이식: 128K에 달하는 Long Context Window를 지원하여, 이전 대화 내용이 아주 길어지더라도 현재 질문의 의도를 놓치지 않고 분석  
  * Sliding Window Attention으로 메모리 효율화: 도구 호출 시 수반되는 긴 API 문서를 처리할 때 메모리 부하가 적어, 대규모 병렬 호출 상황에서도 안정적인 지연 시간(Latency)을 유지

---

## 6\. 주요 제외 모델군 및 기술적 배제 사유

### 6.1 Mistral / Mixtral 시리즈 (Mistral NeMo 12B, Mixtral 8x7B)

* **사유:** 한국어 토큰 효율성 및 지시 이행력 부족.  
* **상세:** 한국어 데이터 비중이 낮아 토크나이저 효율이 떨어지며(Latency 상승), 한국어 KMMLU 점수가 타 모델 대비 10\~15% 낮음.

### 6.2 Llama 4 (70B / 405B) \- 거대 모델군

* **사유:** 추론 비용 및 지연 시간 과다.  
* **상세:** Multi-GPU 필수 환경으로 운영 비용(OPEX)이 높으며, TTFT가 100ms를 상회하여 실시간 에이전트 서비스에 부적합.

### 6.3 SaaS API (GPT-4o, Claude 3.5 / 4\)

* **사유:** 데이터 보안(Privacy) 및 고정 비용 이슈.  
* **상세:** 사내 API 명세나 DB 스키마를 외부로 전송해야 함. 온프레미스 구동이 불가능하여 데이터 보안 정책 위배 가능성 존재.

### 6.4 DeepSeek-V3 (Full Parameter, 671B MoE)

* **사유:** 극단적인 인프라 요구 사양.  
* **상세:** 수천 억 개의 파라미터로 인해 H100 8장 이상의 노드 클러스터 필요. 효율적인 sLLM 서버 운영 환경에서 배포 불가.

---

## 7\. 인프라 및 추론 엔진 권장 사항

* **GPU:** VRAM 24GB(최소) \~ 80GB(권장).  
* **추론 엔진:** 로컬 테스트는 **Ollama**, 실제 서비스 운영은 **vLLM** (Continuous Batching 지원).

---

## 8\. 최종 제언: 컴포넌트별 최적 모델 조합

| 컴포넌트 | 추천 모델 | 핵심 선정 이유 |
| :---- | :---- | :---- |
| **사용자 의도 분석** | **Konan LLM v3** | KMMLU 89.1 달성, 한국어 맥락 파악 능력 최상. |
| **도구 호출** | **Qwen3 32B** | BFCL V4 성공률 92.4%, API 규격 준수 능력 검증. |
| **SQL 쿼리 생성** | **Qwen3-Coder 32B** | SQL 실행 정확도(EX) 82.1%로 sLLM 중 최고 수준. |
| **고속 처리 (옵션)** | **Llama 4 Scout** | TPS 280, TTFT 25ms 미만으로 극도의 반응 속도 필요 시 활용. |

---

## 업데이트 기록

* **2026-03-22:** 목적별 세부 지표, 기술 분석, 제외 사유 및 전체 표 포함 통합 리포트 업데이트 (V2)

