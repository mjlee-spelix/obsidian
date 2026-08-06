---
tags: [프로젝트, dify, AI-Agent, HDD]
status: 초안
date: 2026-04-28
last_updated: 2026-05-26
---
# SPX-Agent 대시보드 - Defect Catalog

> AI에게 구현을 시킬 때 **"이 Harness를 반드시 방어해줘"** 라고 지시하는 결함 패턴 목록.
> 프로젝트 노트의 이슈/리스크 + 지식노트 분석에서 추출함.

## 사용 방법

```
"requirements.md를 읽고 KPI 합산 API를 구현해줘.
 H-DASH-01, H-DASH-05, H-DASH-09를 반드시 방어하고
 각 Harness에 대한 단위 테스트도 함께 작성해줘."
```

---

## 데이터 집계 결함

### H-DASH-01: AppMode별 토큰 이중 카운트

| 항목 | 내용 |
|------|------|
| **상황** | 부서별 토큰 합산 시 `messages`와 `workflow_runs`를 단순 합산하면 ADVANCED_CHAT 앱의 토큰이 두 번 카운트됨 |
| **원인** | ADVANCED_CHAT은 `messages`에도 토큰 저장, `workflow_runs`에도 토큰 저장 — 값은 같음 |
| **영향** | KPI 카드 토큰 수치 부풀림, 부서별 활동 테이블 토큰 왜곡 |
| **방어** | AppMode별 분기: COMPLETION/CHAT/AGENT_CHAT → `messages`만, WORKFLOW → `workflow_runs`만, ADVANCED_CHAT → `messages`만 (기존 프론트엔드 기준 유지). **단 모델 차원 차트(모델별 토큰·모델별 호출 점유)는 좁은 예외** — 모델 *라벨/호출수*는 `workflow_node_executions.process_data`에서 가져옴(ADVANCED_CHAT message는 model NULL이라 메시지 레벨 분해 불가). 호출/토큰 *총량* 집계는 그대로 `messages`. 이중카운트 금지 원칙 유지: 모델 마트 안에서 ADVANCED_CHAT을 message_send/노드 한쪽으로만 카운트 (`app_mode_d <> 'advanced-chat'` 분기). 상세: [[3. 프로젝트/spx-agent/references/workflow-model-classification.md]] §6-1·§7 |
| **테스트** | ADVANCED_CHAT 앱의 토큰이 합산(KPI/부서)에서 1번만 카운트되는지 + 모델 차트의 ADVANCED_CHAT 토큰 합이 message 합과 일치하는지 |
| **관련** | [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]], [[3. 프로젝트/spx-agent/references/workflow-model-classification.md]] |

### H-DASH-02: WORKFLOW 모드 모델별 토큰 분리 불가

| 항목 | 내용 |
|------|------|
| **상황** | 모델 차원 차트 2종(모델별 토큰 + 모델별 호출 점유)에서 모델별 분리 불가. **두 부류**: ① WORKFLOW 앱 — message 행 자체가 없어 차트에서 통째로 누락 ② ADVANCED_CHAT(챗플로우) — message는 있으나 `model_provider`/`model_id`가 **NULL로 생성**되어 '미분류'로 빠짐 |
| **원인** | 모델 정보는 `messages`/`workflow_runs`가 아니라 **모델 보유 노드 3종(`llm`/`parameter-extractor`/`question-classifier`)의 `workflow_node_executions.process_data` JSON**에만 있음. (검증: `message_based_app_generator.py:140-144` — 챗플로우는 `model_provider=None, model_id=None`으로 Message 생성. `generate_task_pipeline.py:958-963` — 챗플로우 message 토큰 = 그래프 전체 노드 usage 합) |
| **영향** | 모델 차트에서 WORKFLOW 누락 + ADVANCED_CHAT은 모두 '미분류' 한 덩어리. **현재 차트의 '미분류' 토큰 정체 = 챗플로우** (워크플로우 아님 — 워크플로우는 아예 빠져 있음) |
| **방어** | **1차(현 구현)**: `spx_mv_model_tokens_daily`가 `action='message_send'`만 집계 + `COALESCE(model_*,'미분류')`. WORKFLOW 누락, ADVANCED_CHAT='미분류'. **2차(B안 채택 방향)**: 모델 차트 2종을 노드 기반 모델 마트(`workflow_node_execute` + `model_id_d IS NOT NULL`)에서 공급. ADVANCED_CHAT은 message_send에서 빼고 노드에서 재분류(§6 UNION). **범위는 모델 차원 2종에 한정** — 부서/앱 차원·canonical 호출수는 message_send 유지(H-DASH-01). process_data 오프로딩 시 일부 노드 모델 누락 가능 |
| **테스트** | (1차) WORKFLOW 토큰 누락·ADVANCED_CHAT이 '미분류'에 포함되는지 (2차/B안) 노드 기반 모델 합 == kpi_calls 챗플로우 합, 모델별 호출 점유 합 ≥ KPI 총 호출량(단위 차이 정상) |
| **관련** | [[3. 프로젝트/spx-agent/references/workflow-model-classification.md]] (소스 검증 + A안/B안), [[4. 지식노트/Dify - workflow_node_executions에서 모델별 토큰 추출.md]] |

### H-DASH-03: 디버깅 실행 데이터 혼입

| 항목 | 내용 |
|------|------|
| **상황** | Dify Studio에서 테스트/디버깅한 실행이 통계에 포함되면 수치 왜곡 |
| **원인** | `messages`는 `invoke_from = 'debugger'`, `workflow_runs`는 `triggered_from = 'debugging'`으로 표시됨 |
| **영향** | 활성 사용자 수, API 호출 수, 토큰 사용량 부풀림 |
| **방어** | 모든 집계 쿼리에 필터 필수: `messages.invoke_from != 'debugger'` + `workflow_runs.triggered_from = 'app-run'` |
| **테스트** | 디버깅 실행 데이터 넣은 뒤 통계에 포함 안 되는지 |
| **관련** | [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]] |

### H-DASH-18: Dify 예외 타입 정보 DB 미보존 → `messages.error` 분류는 ILIKE만 가능

| 항목 | 내용 |
|------|------|
| **상황** | top-error-types 차트(⑱ 부서별 주요 에러 유형) 등 에러 분류 기능 구현 시, `messages.error` 컬럼을 `isinstance` 또는 SDK error code로 분류하려는 시도가 영원히 실패함. `error_code`/`error_type` 컬럼이 없는데 추가 컬럼이 있을 거라 가정하고 ETL을 작성하는 것도 같은 함정. |
| **원인** | Dify의 3단계 에러 변환 경로 마지막 단계(`api/core/app/task_pipeline/based_generate_task_pipeline.py:65-67`)에서 `Exception → str` 변환이 일어남 — `message.error = err_desc`. **이 시점에 `InvokeRateLimitError` 같은 예외 클래스 정보가 완전히 소실되어** 문자열만 DB에 저장됨. Plugin daemon이 보낸 SDK `error.code`(예: `"rate_limit_exceeded"`), `error_type`(예: `"InvokeRateLimitError"`)은 plugin daemon 내부에서 InvokeError 서브클래스 선택에만 소비되고 Dify 백엔드에 도달할 때는 이미 `description` 문자열만 남음. `messages` 테이블에 `error_code`/`error_type` 컬럼 자체가 없고(`api/models/model.py:1418` — `error` LongText 단일), `message_metadata`는 정상 완료 경로에서만 채워짐(에러 시 NULL). `workflow_runs.error` / `workflow_node_executions.error`도 동일 패턴. |
| **영향** | (a) `isinstance` 기반 분류 로직 작성 시도 시 컴파일은 되지만 항상 false → 모든 에러가 "unknown" 버킷으로 빠짐 (b) SDK error code 사용을 전제로 한 마트 설계는 ETL 단계에서 데이터 없음 발견 → 설계 폐기 (c) Dify upstream 변경 또는 collector 보강(옵션 2) 없이는 회피 경로 없음 (d) 향후 누군가 `messages` 테이블에 `error_code` 컬럼 추가 시도하면 Dify 무수정 원칙 위반 |
| **방어** | (a) **분류는 ILIKE 정규식만 사용** — 5~6 룰로 주요 에러 90%+ 커버 가능. 룰 카탈로그 출처: [[3. 프로젝트/spx-agent/references/dify-error-flow.md]] § 6 (b) InvokeError 서브클래스의 `description` 기본값이 안정적 키워드(Rate Limit / Connection / Incorrect API key / Server Unavailable / Bad Request / Quota) 제공 — Auth/Quota는 고정 문자열로 매칭 안전성 최고 (c) **룰 갱신 빈도 임계 도달 시 외부 룰 테이블(옵션 5)로 분리** — ETL 코드 수정 없이 운영 가능 (d) Spec에 "isinstance 분류 금지" + "error_code 컬럼 추가 시도 금지" 명시 |
| **테스트** | (a) 분류 함수에 InvokeError description 6종 입력 → 각각 정확한 카테고리로 분류되는지 (b) SDK 원문 메시지(예: `"Rate limit reached for gpt-4o on..."`) 입력 시에도 `rate_limit`로 분류되는지 (c) `messages.error_code` 컬럼 참조 코드가 존재하지 않는지 grep |
| **관련** | [[3. 프로젝트/spx-agent/references/dify-error-flow.md]] (전체 흐름 + 룰 카탈로그), [[3. 프로젝트/spx-agent/hdd/design.md]] § 2(옵션 D 채택) + § 10(옵션 5 도입 시점 미해결), [[1. Daily/2026-05-12.md]] (VSCode Claude 조사로 확정) |

### H-DASH-04: 레거시 앱 부서 미배정 + 트랜잭션 abort 함정

| 항목 | 내용 |
|------|------|
| **상황** | RBAC `spx_resource_ownership` 테이블(5/4 가정명 `sp_object_ownership`) 도입 전에 만들어진 기존 앱/데이터셋/도구가 부서 배정 없이 존재 |
| **원인** | `spx_resource_ownership`은 신규 테이블이라 기존 데이터에 행이 없음. 더해서 `spx_` 테이블 자체가 미생성 시 `try/except` fallback 진입 시점에 PostgreSQL 트랜잭션이 이미 aborted 상태 |
| **영향** | (a) 부서별 오브젝트 분포, 부서별 활동 테이블에서 기존 앱이 통째로 누락 (b) **첫 쿼리 실패 후 rollback 안 하면 같은 세션 후속 쿼리가 모두 `InFailedSqlTransaction`으로 폭발 → 500 에러** (2026-05-06 dept-objects 발견) |
| **방어** | LEFT JOIN + "미배정" 부서 fallback 처리. **try/except fallback 시 except 첫 줄에 `db.session.rollback()` 필수** — 안 하면 PostgreSQL 트랜잭션 abort 상태에서 후속 쿼리/fallback SQL 모두 거부됨. KPI 총 오브젝트 수는 `apps` 테이블 기준 COUNT와 `spx_resource_ownership` COUNT 교차 검증. 패턴: ```python try: ... except Exception: db.session.rollback(); logger.exception("..."); return [] ``` |
| **테스트** | (a) `spx_resource_ownership`에 없는 앱이 "미배정"으로 집계에 포함되는지 (b) **`spx_` 테이블 미생성 환경에서 service 메서드 순차 호출 시 두 번째 메서드도 정상 응답하는지** (트랜잭션 회복 검증) |
| **관련** | [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]], [[4. 지식노트/PostgreSQL - InFailedSqlTransaction과 SQLAlchemy fallback 패턴]] |

---

## 기간·데이터 결함

### H-DASH-05: Celery Beat 데이터 삭제로 통계 누락

| 항목 | 내용 |
|------|------|
| **상황** | `clean_messages`(매일 4시), `clean_workflow_runlogs`(매일 2시)가 보존 기간 지난 데이터를 삭제 |
| **원인** | 대시보드 기간 선택기에서 보존 기간보다 긴 기간을 선택하면 과거 데이터가 없음 |
| **영향** | KPI 증감률(%) 왜곡 — 이전 기간 데이터가 삭제되어 증감 비교 불가, 수치가 급증한 것처럼 보임 |
| **방어** | 보존 기간 설정값 확인 → 기간 선택기 최대 범위 제한, 또는 이전 기간 데이터 없을 때 증감률 "N/A" 표시 |
| **테스트** | 이전 기간 데이터가 0일 때 증감률이 "N/A"로 표시되는지 |
| **관련** | [[4. 지식노트/Dify - Celery Beat 정기 백그라운드 작업.md]] |

### H-DASH-06: 비동기 저장 지연으로 최신 데이터 누락

| 항목      | 내용                                                                           |
| ------- | ---------------------------------------------------------------------------- |
| **상황**  | WORKFLOW/ADVANCED_CHAT의 `workflow_runs`가 Celery 비동기로 저장되므로 방금 실행한 것이 바로 안 보임 |
| **원인**  | `CeleryRepository`가 Redis 큐에 `.delay()`로 넣고, worker 컨테이너가 꺼내서 DB 커밋. 실패 시 60초 간격 최대 3회 재시도 |
| **지연 시간** | 코드상 측정 로직 없음 — 환경(Redis/worker 부하)에 따라 다름. worker 유휴 시 수백 ms ~ 수 초, 부하 시 수십 초까지 가능. 정확한 수치는 실제 환경 테스트 필요 |
| **영향**  | 사용자가 "방금 실행했는데 대시보드에 안 나와요" 문의                                               |
| **방어**  | UI에 "데이터 반영에 수 분이 소요될 수 있습니다" 안내 또는 새로고침 시 최신 커밋 대기                          |
| **테스트** | (기능 테스트보다 UX 안내 여부 확인)                                                       |
| **관련**  | [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md]]                             |

---

## 계산·표시 결함

### H-DASH-07: 증감률 분모 0 → NaN/Infinity

| 항목 | 내용 |
|------|------|
| **상황** | KPI 카드 증감률(%) 계산 시 이전 기간 값이 0이면 나눗셈 오류 |
| **원인** | `(현재 - 이전) / 이전 × 100` 공식에서 이전 = 0 |
| **영향** | UI에 NaN%, Infinity% 표시 또는 JavaScript 에러 |
| **방어** | 이전 = 0이고 현재 > 0 → "+신규" 표시, 이전 = 0이고 현재 = 0 → "-" 표시, 그 외 정상 계산 |
| **테스트** | 이전 기간 0, 현재 기간 양수/0 각각에서 UI 깨지지 않는지 |

### H-DASH-08: 활성 사용자 COUNT에서 NULL 포함

| 항목 | 내용 |
|------|------|
| **상황** | `messages.from_end_user_id`가 NULL인 행이 `COUNT(DISTINCT)` 결과를 왜곡 |
| **원인** | API 호출(서비스 토큰 방식) 시 end_user가 없을 수 있음 |
| **영향** | 활성 사용자 수가 실제보다 적거나 NULL이 한 명으로 카운트됨 |
| **방어** | `WHERE from_end_user_id IS NOT NULL` 필터 추가 |
| **테스트** | NULL end_user가 있는 상태에서 활성 사용자 수가 정확한지 |

### H-DASH-09: 로컬 모델 표시 구분

| 항목      | 내용                                                                                        |
| ------- | ----------------------------------------------------------------------------------------- |
| **상황**  | 모델별 토큰 차트에서 로컬 모델(Llama, Gemma 등)과 외부 API 모델(GPT, Claude)을 구분해야 함                         |
| **원인**  | 화면 설계에 "Llama 3 (로컬)" 표시 → `model_provider`로 로컬/외부 판별 로직 필요                               |
| **영향**  | 구분 없으면 내부 비용 0원 모델과 외부 유료 모델이 섞여 비용 판단 어려움                                                |
| **방어**  | 플러그인 경로 마지막 세그먼트 추출(`rsplit("/",1)[-1]`) 후 `LOCAL_PROVIDERS` frozenset 매칭(방법 A). frozenset에 `vllm` 추가. 2곳 중복 정의는 공유 상수화 |
| **테스트** | 로컬 provider의 모델에 "(로컬)" 라벨이 붙는지. **시드/픽스처를 운영과 동일한 namespaced 형식(`langgenius/ollama/ollama`)으로 작성** — bare 시드는 green이어도 운영 버그 못 잡음 |
| **참고**  | `messages.model_provider` 필드에 provider명 저장 확인. 신호 비교·DB 실증·권고안 전문: [[3. 프로젝트/spx-agent/references/local-provider-classification.md]] · 구현 체크리스트: [[3. 프로젝트/spx-agent/hdd/specs/tasks/local-provider-classification.md]] |

> **2026-06-10 조사 확정 (namespaced 미스매치 버그)**
> - **버그 본질**: 운영 `model_provider`는 **플러그인 경로 형식**(`langgenius/ollama/ollama`, `yangyaofei/vllm/vllm`)인데 현재 `LOCAL_PROVIDERS = {"ollama","xinference","localai"}`는 **bare 이름** → 운영에서 "(로컬)" 라벨 **0건**(ollama조차 미매칭).
> - **형식 근거 (코드 체인 — 변환 없는 pass-through)**: Dify `messages.model_provider` → `messages.ts` collector `details.modelProvider` → `spx_audit_events.model_provider_d`(Generated Column) → `spx_mv_audit_enriched.model_provider`(alias) → `spx_mv_model_tokens_daily.model_provider`(COALESCE). 어느 단계도 포맷 변환 없음 → mart 형식 = Dify 원본 형식 = 플러그인 경로.
> - **DB에 로컬 플래그 없음**: `is_local`/`deployment_type`/`category` 직접 필드 부재(조사 3.2). `provider_type`은 custom/system(자격증명 범위)이라 로컬 구분 아님. 유일한 진짜 신호(endpoint 사설 IP)는 RSA 암호화 + mart에 없음 → 비용 큼(비채택).
> - **사용처 2곳(공유 상수화 대상)**: `dashboard_model_tokens_service.py:19` `LOCAL_PROVIDERS` / `dashboard_drill_calls_service.py:49` `_LOCAL_PROVIDERS`.
> - **openai_api_compatible**: 로컬/원격 동명 → 이름으로 판정 불가. 현 방침 = **미표시 감수**(향후 필요 시 DB/env allowlist로 승격).
> - 구현은 **PM 결정 4건(권고안 §9) 사인 후** 착수. 결정 항목 = vllm 추가 범위 / 방법 A vs B / mart 형식 확인 / openai_api_compatible 방침.

---

## 성능 결함

### H-DASH-10: 부서별 활동 쿼리 성능

| 항목      | 내용                                                                                          |
| ------- | ------------------------------------------------------------------------------------------- |
| **상황**  | 부서별 활동 테이블의 집계 쿼리가 `spx_departments → spx_resource_ownership → apps → messages/workflow_runs` 다단 JOIN |
| **원인**  | 가장 복잡한 쿼리이고, 부서 수 × 앱 수 × 메시지 수만큼 스캔                                                        |
| **영향**  | 쿼리 타임아웃, 대시보드 로딩 지연                                                                         |
| **방어**  | Redis 캐시(staleTime 5분) + 부서별 집계는 서브쿼리/CTE로 단계 분리 + 필요 시 인덱스 추가                              |
| **테스트** | 부서 10개, 앱 100개, 메시지 10만건 기준 쿼리 응답 2초 이내                                                     |

### H-DASH-11: workflow_node_executions JSON 파싱 성능

| 항목 | 내용 |
|------|------|
| **상황** | 모델별 토큰을 `workflow_node_executions.process_data` JSON에서 추출하면 인덱스 못 탐 |
| **원인** | `::json->>'key'`는 전체 행 스캔 + 노드 수 × 실행 수만큼 데이터 |
| **영향** | 30일 범위에서 수초~수십초 소요 가능 |
| **방어** | 이 쿼리 쓸 경우 반드시 Redis 캐시 (TTL 5~15분) + 결과 건수 제한 + 비동기 로딩 |
| **테스트** | (2차 구현 시) 캐시 미적용 vs 적용 응답시간 비교 |
| **관련** | [[4. 지식노트/Dify - workflow_node_executions에서 모델별 토큰 추출.md]] |

### H-DASH-12: CSV 내보내기 대량 데이터

| 항목 | 내용 |
|------|------|
| **상황** | 부서별 활동 테이블 CSV 내보내기 시 대량 데이터를 한 번에 메모리에 올림 |
| **원인** | 부서 × 기간이 길면 행 수 폭증 |
| **영향** | 서버 메모리 급증, OOM |
| **방어** | 스트리밍 응답 or 행 수 제한(최대 1000행) + 제한 초과 시 안내 |
| **테스트** | 1000행 초과 요청 시 제한 동작 확인 |

---

## 연동·충돌 결함

### H-DASH-13: RBAC 테이블 스키마 변경 영향

| 항목      | 내용                                                                       |
| ------- | ------------------------------------------------------------------------ |
| **상황**  | 김이사님(결정) / 권대리님(구현) RBAC 테이블(`departments`, `resource_ownership` 등) 스키마가 변경되면 대시보드 쿼리 전체 영향 |
| **원인**  | 대시보드가 RBAC 테이블에 직접 의존                                                    |
| **영향**  | 컬럼명/타입 변경 → 쿼리 에러, FK 구조 변경 → JOIN 실패                                    |
| **방어**  | 대시보드 쿼리 코드에서 DB 컬럼명을 직접 쓰지 않고, **SQLAlchemy 모델 클래스**(예: `Department.name`, `ResourceOwnership.owner_department_id`)를 통해 접근. 이러면 권대리님이 컬럼명을 바꿔도 모델 클래스 파일 하나만 고치면 되고, 쿼리/서비스 코드는 안 건드려도 됨. HDD 문서에 변경 영향 규칙 명시 |
| **회피책 작동 사례 (2026-05-06)** | 5/4 spec은 `sp_object_ownership.object_type/object_id/department_id` 가정 → 실제는 `resource_ownership.resource_type/resource_id/owner_department_id`. 코드 어디에 sp_ 컬럼명이 박혀있었으면 갈아끼우기 비용 큼. 다행히 spec 단계에서 어긋남 발견(H-DASH-17) → 백엔드 service 작성 전 정정 → SQLAlchemy 모델 레이어에서 흡수 가능. **"회피책이 작동하려면 모델 레이어를 거쳐야 함" 입증 사례**. |
| **테스트** | (코드 리뷰 레벨) 모델 파일 변경 시 쿼리/서비스 코드 자동 점검                                    |
| **관련**  | [[3. 프로젝트/SPX-Agent 하네스 설계.md|하네스 설계]], [[3. 프로젝트/spx-agent/references/rbac-schema.md]] (2026-05-06 갱신), [[3. 프로젝트/spx-agent/hdd/defect-catalog.md#H-DASH-17]] (관찰 부족 함정 — 본 결함의 발견 트리거) |

### H-DASH-14: ~~users vs accounts 테이블 불일치~~ → 2026-05-06 자동 해소

| 항목 | 내용 |
|------|------|
| **2026-05-06 해소 사실** | 권대리님의 RBAC 영역 발견(H-DASH-17 트리거)으로 별도 `sp_users` 테이블 자체가 **없음** 확인. `department_members.account_id` 컬럼이 Dify `accounts.id`를 직접 참조하는 구조 → 매핑 키 별도 결정 불필요. 이 결함 패턴 자체가 **구조적으로 불가능**한 상태. |
| **결과 변경** | 부서별 사용자 메트릭 (dept-dau, dept-user-activity-table) 즉시 작업 가능. 단 `department_members` 데이터 자체가 채워져야 동작 (mock fixture 또는 Keycloak SSO upsert 구현 후). |
| **historical 기록** | (5/4 spec 시점 가정) RBAC `sp_users` 테이블과 Dify `accounts` 테이블이 별개라고 가정 → 사용자 기준 JOIN 시 매핑 키 미정 우려. **실제는 권대리님이 `account_id` 직접 매핑으로 설계해서 우려 자체가 사라짐**. |
| **남은 의존** | `department_members` 실 데이터 채우는 시점 (Keycloak SSO upsert 또는 김이사님/태영님 화면에서 수동 입력). 우리 mock 단계에선 fixture INSERT로 검증. |
| **관련** | [[3. 프로젝트/spx-agent/references/rbac-schema.md]], H-DASH-17 (관찰 부족 함정) |

### H-DASH-15: ~~설정 모달 사이드바 충돌~~ → 2026-05-13 종결

| 항목      | 내용                                                                   |
| ------- | -------------------------------------------------------------------- |
| **상황**  | 설정 모달 사이드바를 본인(대시보드) + 김이사님(부서/사용자관리) + 승랑님(감사로그)이 각자 수정             |
| **종결 사유** | 2026-05-13 대시보드 마운트 위치 변경 (설정 모달 탭 → `/dashboard` 톱레벨 라우트). 우리 영역이 설정 모달에서 빠지면서 사이드바 충돌 의존 자체 소멸 |
| **참조**  | SESSION_HISTORY 2026-05-13 변경 이력 |

### H-DASH-19: 모달 마운트 제약 폐기 (2026-05-13)

| 항목 | 내용 |
|------|------|
| **상태** | 🔒 폐기 (2026-05-13) — 함정 자체가 사라짐 |
| **과거 패턴** | 대시보드를 설정 모달 탭(`account-setting/dashboard-page/`)으로 마운트 시 제약: 자체 `<h1>` 금지 / URL state 금지 / ESC 자체 핸들링 금지 / `dashboard-controls`를 모달 헤더 우측 슬롯에 배치 / dialog 환경 충돌 위험 |
| **폐기 사유** | 2026-05-13 라우트 이동(`/dashboard`) + 사용자 범위 전체 공개로 모달 마운트 패턴 자체 폐기. 라우트 페이지 환경에서 위 제약 모두 해제됨 (자체 h1 작성 가능 / URL query string 사용 가능 / ESC 자유 활용 / 페이지 헤더 슬롯 사용) |
| **함정 방지** | 신규 합류자가 "옛날 spec에 모달 제약 박혀있던데..."라고 의문 가질 수 있음 → 본 항목으로 빠른 답 제공. 현재 모든 spec/CLAUDE.md/architecture는 라우트 페이지 전제 |
| **참조** | SESSION_HISTORY 2026-05-13 변경 이력 (라우트/사이드바 결정 확정 항목) |

### H-DASH-20: 차트 드로어 4종 보류 (2026-05-13)

| 항목 | 내용 |
|------|------|
| **상태** | 🔒 보류 (2026-05-13 이사님 결정) — 함정 아닌 결정 사항이지만 spec 참조용으로 박음 |
| **과거 패턴** | 좌하 차트 클릭 → 우측 차트 드로어 표시 (`chart-drawer` spec 3파일에 박힘). 4종 가치 검증 결과 임원 의견 수렴 예정이었음 |
| **보류 사유** | (1) 정보 한 단계 깊게 보는 게 사용자에게 의미 있는지 재판단 필요 (2) 향후 관리자 전용으로 만들 가능성 |
| **현재 동작** | 좌하 차트 클릭 자체 제거 — 차트만 표시. drill-through 안의 차트 모두 클릭 비활성 |
| **재검토 시점** | 사용자 피드백 + 관리자 권한 분리 정책 확정 후 |
| **참조** | `chart-drawer` spec 3파일 (보류 헤더만 박힘, 본문 보존) / SESSION_HISTORY 2026-05-13 차트 드로어 보류 항목 |

### H-DASH-21: 컨텍스트바 보류 결정 (2026-05-22)

| 항목 | 내용 |
|------|------|
| **상태** | 🔒 보류 (2026-05-22) — H-DASH-20(chart-drawer)과 자매 패턴. 함정 아닌 결정 사항이지만 spec/코드 참조용으로 박음 |
| **상황** | 모달 마운트 폐기(5/13 H-DASH-19) + URL query string 단일 진실 채택 후, 컨텍스트바의 "현재 선택 메트릭 표시 + 해제 버튼" 책임이 KPI 카드 active ring + ESC 토글로 자연 분산 흡수됨 → 컨텍스트바 렌더링 잉여 |
| **원인** | 5/22 화면 검증 결과 KPI 카드 active ring + ESC 토글만으로 drill-through 진입/해제 UX 충분. 컨텍스트바 본문(기간 필터/캐시 안내 등) 부가 정보는 다른 위치(`DashboardControls` 호버 툴팁 후보)로 흡수 검토 |
| **현재 동작** | (a) `page.tsx`에서 `<ContextBar>` 렌더링 제거됨 — `grep -n "ContextBar" web/app/(commonLayout)/dashboard/page.tsx` → 0건 (b) 컴포넌트 코드(`web/app/components/admin/context-bar/`)는 **보존** (재도입 base) (c) spec 3파일(`requirements`/`design`/`tasks/context-bar.md`)에 `🔒 보류 (2026-05-22)` 헤더 박힘, 본문 base 보존 |
| **영향** | (a) 컴포넌트 코드 살아있어 미사용 lint 경고 가능 (b) 다른 작업자가 "코드 살아있네" → 즉흥 부활 시 drill-through UX 회귀 위험 (c) 추후 재도입 결정 시 spec base 그대로 활용 가능 |
| **방어** | (a) `page.tsx` import/렌더링 0건 **유지** 의무 (b) 컴포넌트 코드 보존, 삭제 금지 (c) spec 3파일 `🔒 보류 (2026-05-22)` 헤더 유지 (d) **재도입 트리거는 PM 결정 동반** — 즉흥 부활 금지 |
| **테스트** | (a) `grep -n "ContextBar" web/app/(commonLayout)/dashboard/page.tsx` → 0건 (b) `ls web/app/components/admin/context-bar/` → 디렉토리 존재 (보존 검증) (c) spec 3파일 frontmatter `status: "🔒 보류 (2026-05-22)"` 박힘 |
| **재검토 시점** | drill-through 메트릭 가짓수 증가 / 추가 필터(부서·모델 등) 도입 결정 시 — PM 동반 |
| **참조** | H-DASH-20 (chart-drawer 5/13 보류) 자매 결함 / H-DASH-19 (모달 마운트 폐기 — 본 결정의 출처) / `context-bar.md` 보류 헤더 3종 / SESSION_HISTORY 2026-05-22 § 차트/표 UI 개선 |

### H-DASH-17: 외부 의존 명세를 spec 추정으로 박는 함정 (관찰 부족)

| 항목 | 내용 |
|------|------|
| **상황** | 외부 의존 영역(다른 분 작업 영역)의 테이블/API/컬럼 명세를 **추정으로 spec에 박고**, 실제 환경 직접 관찰을 생략. 의존 영역 명명/구조가 추정과 다를 경우 spec/코드/SQL 모두 어긋남이 영원히 발견 안 됨 (이름이 안 맞으면 grep으로도 못 잡음). |
| **원인** | (a) Spec 작성 시점에 의존 영역 결정이 안 끝났거나 미공유 → 추정이 spec에 박힘 (b) 추정값이 자체 컨벤션 가정(예: "회사 표준 = sp_ prefix")에 기반하면 더 위험 — 회사가 다른 표준 쓰면 영원히 어긋남 (c) 추정 후 실 환경 검증 게이트 부재 → 코드 작성 시점까지 잠복 (d) 의존 영역의 코드/DB가 사실 우리 환경에 이미 존재해도 이름이 안 맞으면 안 보임 |
| **영향** | (a) Spec 전체에 잘못된 명명 박힘 (b) 그 spec 따라 작성된 코드/SQL/모델 모두 잘못됨 (c) 외부 의존 가용성을 잘못 인식 — "그분 ETA 대기" 잘못된 블로커 (d) 갱신 비용 큼 — references/spec/코드/SQL 모두 정정 필요 (e) 다른 사람한테 잘못된 정보 공유 위험 (f) **2026-05-06 사례**: 5/4 spec에 `sp_*` prefix 가정 → 5/6 RBAC 발견까지 "RBAC = 김이사님 ETA 대기" 잘못 인식 → 실제는 우리 환경 DB에 `departments`/`resource_ownership` 등 5종 테이블 영속 잔존 |
| **방어** | **외부 의존 가용성 검증의 3단계 패턴 의무 적용**: (1) **코드 검증** — `git ls-tree origin/<their-branch>`, `git show origin/<br>:<path>`, `git grep` (2) **운영 인프라 검증** — `docker compose exec db psql -c "\dt"`, `\d <table>`, `SELECT count(*)` (3) **사람 검증** — 슬랙으로 명세/명명 확정 받기. 1→2→3 순서로 비용 적게. 1번/2번에서 답 나오면 사람한테 안 물어봐도 됨. **Spec 작성 시점에 frontmatter `external_dependency_observed_at:` 필드 박아 검증 시점 기록 권장**. |
| **2026-05-06 발견 트리거** | 사용자가 "rbac 테이블은 rbac 브랜치에 있지 않을까?" 의문 제기 → `git show origin/feat/rbac:api/models/sp_department.py` 안 나옴 → "DB에 있나?" 추가 의문 → `\dt` 직접 실행 → `departments` 외 5종 발견. 사용자의 단순한 의문 자체가 **방어 패턴의 핵심 단계**였음. |
| **테스트** | (a) 신규 spec 작성 시 외부 의존 영역의 테이블/모델/API가 frontmatter `external_dependency:` 필드로 명시 (b) 그 의존 영역의 실 환경 검증 여부 박힘 (`observed_in_db: true` 또는 `recorded_from: <commit_hash>`) (c) 검증 없이 작성된 spec은 quality-criteria § 5 게이트 통과 X |
| **관련** | [[3. 프로젝트/spx-agent/references/rbac-schema.md]] (2026-05-06 정정 사례), [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (5/6 변경 이력), H-DASH-13 (이 결함의 회피책이 작동하려면 모델 레이어 거쳐야 함), H-DASH-16 (spec과 화면 설계 어긋남 — 비슷한 self-review 함정) |

### H-DASH-16: Spec과 화면 설계 이미지/PDF 어긋남 (자기 검토 함정)

| 항목 | 내용 |
|------|------|
| **상황** | spec 작성 시점에 화면 설계 이미지/PDF를 보면서 작성하지만, 본인이 작성하고 본인이 검토하는 구조라 **세부 어긋남이 spec 단계에서 잡히지 않고 구현 후 사용자 시각 검토 시점에 발견**됨. requirements는 일반론(예: "위치/행 수 유지")으로 정확하게 박혀있어도 design 단계에서 컴포넌트 매핑(차트 타입/표 vs 차트/행 단위)이 화면 설계와 어긋날 수 있음. |
| **원인** | (a) 본인이 작성하고 본인이 검토 → 같은 추상 수준에 갇혀 자기 의도와 산출물 차이를 인지 못 함 (b) 화면 설계가 외부 산출물(이미지/PDF)인데 spec은 텍스트 → 추상 수준이 다른데 1:1 대조 게이트가 없음 (c) requirements ↔ design ↔ tasks의 내부 정합성 게이트도 없음 — design이 requirements 명시 제약을 위반해도 통과 |
| **영향** | (a) 잘못된 spec 기반으로 구현 → 컴포넌트 4개 작성 후 폐기 + 4개 추가 작성 (5/4→5/6 사례) (b) 사용자 시각 검토에서 발견 = 가시 산출물이 김이사님/PM 검토 시점에 노출 가능 (c) spec 문서 신뢰도 자체 흔들림 — "spec 따라 만든 게 왜 어긋나지?" |
| **방어** | (a) **`hdd/quality-criteria.md § 5` Spec 작성 검증 게이트** 통과 필수 — 외부 정합성(이미지 1:1 대조) + 내부 정합성(spec 3파일 상호 검증) (b) spec frontmatter에 `design_image:` / `reference_pdf:` 필드 존재 시 **구현 시작 전 의무 통과** (c) 자기 검토 한계 인지 — 본인 검토 외에 외부 산출물 직접 비교 / AI 에이전트 위임 / 시간차 검토 중 하나 추가 (d) 화면 설계의 "고정된 N개 영역", "위치/행 수 유지" 같은 동작 명세는 spec 본문에 인용 박아 위반 즉시 발견 가능 |
| **테스트** | (a) spec 작성 직후: frontmatter 이미지 경로 → 실 이미지 읽기 → 컴포넌트 매핑 1:1 대조 (b) requirements ↔ design ↔ tasks 메트릭 카탈로그 행 수 일치 (c) 정정 후 회귀: 같은 spec 작성 패턴으로 다른 컴포넌트 작업 시 게이트 통과 여부 확인 |
| **관련** | [[3. 프로젝트/spx-agent/hdd/quality-criteria.md]] § 5, [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (2026-05-06 H-DASH-16 정식 등록) |

---

## 빌드/환경 결함

> 코드 결함이 아닌 **개발 환경 자체의 함정**. 검증된 패턴(실 발생 + 재발)이라 정식 등록.

### H-ENV-01: Linux .venv lib64 symlink가 Windows 호스트에서 깨짐 → BuildKit 컨텍스트 스캔 실패

| 항목 | 내용 |
|------|------|
| **상황** | `docker compose build` 실행 시 `error from sender: open ...\.venv\lib64: The file cannot be accessed by the system` 에러로 빌드 중단 |
| **원인** | Dify 플러그인 컨테이너가 Linux Python venv를 생성하면서 `lib64 -> lib` symlink를 만듦. 이 폴더가 Windows 호스트에 마운트되면 깨진 reparse point로 보임. **BuildKit은 컨텍스트 sender 단계에서 모든 파일을 stat하고, 그 다음에 `.dockerignore`를 적용**하므로 깨진 link에서 폭발 — `.dockerignore`에 `docker/volumes/` 패턴이 있어도 무력 |
| **영향** | 빌드 자체 불가능. 새 플러그인 설치(예: google_drive)할 때마다 새 깨진 symlink 생성 → **재발성 결함** |
| **방어** | 빌드 전 깨진 symlink 자동 정리. 두 가지 사용 패턴: (a) `.\scripts\spx-clean-broken-symlinks.ps1` 단독 실행 (b) `.\scripts\spx-build.ps1 [args]` wrapper로 빌드 — 정리 후 자동으로 `docker compose build` 호출. bash variant도 동일 위치 제공. **Dify Makefile 무수정 원칙상 별도 wrapper 사용** |
| **테스트** | (a) 빌드 직전 `find docker/volumes -name lib64 -type l` 결과 0건 확인 (b) 새 플러그인 설치 후 첫 빌드 시 wrapper 동작 검증 |
| **관련** | [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (2026-04-30 첫 발생, 2026-05-04 재발), Windows reparse point 동작 |

### H-ENV-02: api/worker 서비스에 `build:` 지시 없어 호스트 코드 변경 미반영

| 항목 | 내용 |
|------|------|
| **상황** | api/worker 컨테이너가 새 모듈을 import 못하거나, 코드 변경 후 재시작·재빌드해도 stale 코드 동작. 예: `ImportError: cannot import name 'dashboard' from 'controllers.console.admin'` (실제로는 호스트에 admin/ 폴더 정상 존재) |
| **원인** | `docker/docker-compose.yaml`의 api/worker 서비스가 `image: langgenius/dify-api:1.13.3`만 지정하고 `build:` 지시문 없음 → 레지스트리에서 이미지를 받아 그대로 사용. 호스트 코드는 컨테이너에 반영 경로가 없음. `docker compose build api`도 `No services to build`로 no-op. (web 서비스만 `build:` 있음) |
| **영향** | 우리 신규 모듈(`api/services/admin/`, `api/controllers/console/admin/dashboard.py` 등)이 컨테이너에서 동작 안 함. import 에러로 컨테이너 크래시 루프 가능 |
| **방어** | `docker/docker-compose.override.yaml` 신설로 우리 변경 디렉토리만 부분 volume mount. `controllers/`, `services/`, `tests/` 3폴더만 마운트 (컨테이너 내장 `.venv`/`models`/`libs` 보호). Dify의 docker-compose.yaml 무수정. Docker compose가 자동으로 두 파일 머지. |
| **테스트** | (a) `docker compose up -d api` 후 컨테이너 status `Up` (Restarting 아님) (b) 호스트에서 `services/admin/utils.py` 한 줄 수정 → 컨테이너 재시작 → 변경 반영 확인 |
| **관련** | [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (2026-05-04 mock 컨트롤러 부팅 실패), `docker/docker-compose.override.yaml` |

### H-ENV-04: Windows Git Bash 멀티바이트 인자 깨짐 (한글 인자 mojibake)

| 항목 | 내용 |
|------|------|
| **상황** | `scripts/sync_dept_keycloak.sh` 같은 bash 스크립트를 Windows Git Bash에서 실행 시 한글 부서명(`code`/`name`)이 cp949로 전달되어 Keycloak API에 `?` 또는 mojibake로 박힘. 5/22 Keycloak 부서 동기화 작업 중 발견 |
| **원인** | Windows Git Bash의 MSYS layer가 환경/인자 변환 시 UTF-8 강제 안 함. 한글 인자가 cp949(`euc-kr` 변종)로 호출 프로세스에 전달되고, 받는 쪽(curl/python)이 UTF-8 가정 시 깨짐. PowerShell 5.1 환경의 한글 인코딩 함정(`delegation-standard.md § 5`)의 자매 패턴 |
| **영향** | (a) Keycloak 그룹명/사용자 firstName 등 한글 필드 깨짐 → DB `code`/`name`과 매칭 실패 (b) 매칭 우선순위 1단계(UUID) 통과 후 2단계(code) 매칭이 mojibake로 fail → 신규 부서 잘못 생성 (c) 디버깅 시 로그에 박힌 mojibake 보고 원인 추적 어려움 |
| **방어** | (a) **한글 인자 동반 스크립트는 python으로 전환** (5/22 채택 — `sync_mock_users_keycloak.py`) (b) bash 유지 시 `iconv -f cp949 -t utf-8` 변환 또는 `chcp 65001` 선행 — 단 fragile, python 권장 (c) 위임 PROMPT.md `§ 5 한글 인코딩` 가드 인지 → 한글 인자 bash echo/sed 금지 (d) python 스크립트는 `# -*- coding: utf-8 -*-` 선언 + `sys.stdout.reconfigure(encoding='utf-8')` 명시 |
| **테스트** | (a) `python sync_mock_users_keycloak.py --dry-run \| grep "부서"` 한글 출력 정상 (mojibake 0) (b) 실행 후 Keycloak admin UI에서 그룹명/사용자명 한글 정상 표시 (c) DB `spx_departments.name` ↔ Keycloak group name 1:1 매칭 100% |
| **관련** | SESSION_HISTORY 2026-05-22 § 트러블슈팅, `delegation-standard.md § 5` PowerShell 한글 인코딩(자매 함정), [[4. 지식노트/Windows Git Bash 멀티바이트 인자 깨짐]] (작성 예정) |

### H-ENV-03: 호스트 Node 버전 drift로 vitest/ESLint hook 부팅 불가

| 항목 | 내용 |
|------|------|
| **상황** | `pnpm test` 또는 commit 시 ESLint hook 실행이 모듈 로드 단계에서 실패. 에러: `The requested module 'node:fs/promises' does not provide an export named 'glob'`. **vitest 자체가 부팅 못 함** + **commit 자체가 차단됨** |
| **원인** | `vinext@0.0.40` 패키지가 Node 22+의 `fs.glob` API를 호출. 호스트 Node가 22 미만이면 import 단계에서 폭발. 회사 표준 `.nvmrc=22`가 명시돼있으나 호스트 Node 버전이 따라가지 않으면 발생 — **개발자 ↔ 회사 표준 환경 drift** |
| **영향** | (a) 프론트 단위 테스트 부팅 자체 불가 (5/4 첫 발견) (b) ESLint pre-commit hook이 `vite.config.ts` 로드 실패로 commit 차단 (5/6 재발) (c) `pnpm dev`/`pnpm build`도 같은 영향 가능. **재발성 확정** |
| **방어** | (a) 신규 셋업 시 `nvm use 22` 또는 `nvm install 22` 강제 (`.nvmrc` 활용 권장) (b) `corepack enable` 후 `pnpm install`로 환경 일치 (c) `architecture.md § 9.2` 셋업 체크리스트 따름 (d) **자동화 검토**: `volta` / `mise` / shell hook으로 `.nvmrc` 자동 활성화 |
| **테스트** | (a) `node --version` → v22.x.x 출력 (b) `pnpm vitest run app/components/admin/dashboard-controls` → 6/6 통과 (1.67s) (c) `git commit` 시 `Running ESLint on web module` 출력 + 정상 진행 |
| **관련** | [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (2026-05-04 vitest 부팅 불가 1차, 2026-05-06 ESLint hook 재발 + Node 22 업그레이드 해소), `.nvmrc`, [[4. 지식노트/Node - corepack과 패키지 매니저 버전 통일]] |

---

## 마트/데이터 결함 (2026-05-19 신설 카테고리)

### H-MART-01: enriched view 비-UUID actor 캐스트 함정

| 항목 | 내용 |
|------|------|
| **상황** | `spx_mv_audit_enriched` MView refresh 또는 downstream JOIN에서 `invalid input syntax for type uuid: "system"` 등 캐스트 에러로 폭발. mock 데이터에 account/end_user만 있던 시점엔 검증 통과 → 운영 진입 후 잠복 폭발 가능 |
| **원인** | `spx_audit_events.actor_id`는 `actor_type IN ('account','end_user','api','system')` 4종인데 **api/system은 비-UUID 문자열**(예: `'chatbot-public-001'`, `'system'`). enriched view SELECT에서 `actor_id::uuid` 직접 캐스트하거나, downstream에서 `actor_id`를 다른 UUID 컬럼(`department_members.account_id` 등)과 직접 비교하면 PostgreSQL이 `text = uuid` operator 에러 또는 캐스트 실패 |
| **영향** | (a) enriched MView refresh 실패 → mart lag 영구 증가 → 대시보드 KPI 정합성 파괴 (b) downstream Layer 2 mat refresh chain 연쇄 실패 (c) collector polling cycle은 정상 동작하나 마트만 깨져 "데이터는 들어오는데 화면에 안 나옴" 사용자 신고 패턴 |
| **방어** | (a) **SELECT 단계 가드**: `CASE WHEN actor_type IN ('account','end_user') THEN actor_id::uuid ELSE NULL END AS actor_id` 패턴 (b) **JOIN 단계 가드**: `department_members` LEFT JOIN ON절에 `dm.account_id = CASE WHEN ae.actor_type='account' THEN ae.actor_id::uuid END` (end_user/api/system은 NULL → 매칭 안 됨 → 자연 LEFT JOIN miss → sentinel UUID 매핑) (c) **테스트 fixture에 actor_type 4종 모두 포함** 의무화 — mock에 account/end_user만 있으면 가드 빠져도 검증 통과해 잠복 |
| **테스트** | (a) fixture에 `actor_type='api'` + `actor_id='chatbot-public-001'` 행 1건 이상 + `actor_type='system'` + `actor_id='system'` 행 1건 이상 (b) `REFRESH MATERIALIZED VIEW public.spx_mv_audit_enriched` 직접 실행 시 에러 없이 완료 (c) Layer 2 `spx_mv_kpi_calls_daily`에 해당 actor_type 행의 `actor_dept_id`가 sentinel UUID(`00000000-...`)로 박힘 확인 |
| **관련** | [[3. 프로젝트/spx-agent/hdd/design.md]] § 2.5.4 "5/18 적용 보정" 블록, 마이그레이션 `20260518100000_actor_id_non_uuid_safe`, [[3. 프로젝트/spx-agent/references/audit-schema.md]] § 1.1 (actor_id 컬럼 actor_type별 형식 정의) |

---

## 인프라/컨테이너 결함 (2026-05-19 신설 카테고리)

> 기존 H-ENV-XX는 빌드 / .venv / Node 등 **개발 환경**. H-INFRA-XX는 Shadow DB / docker volume / dist 캐시 등 **운영 인프라**.

### H-INFRA-01: Shadow DB 부채 — Prisma migrate dev 재생 불가 누적

| 항목 | 내용 |
|------|------|
| **상황** | Prisma 마이그레이션을 추가한 뒤 `prisma migrate dev` 실행 시 Shadow DB 재생 단계에서 폭발. 우회로 `mkdir + resolve --applied`로 정합만 맞추면 매 마이그레이션마다 부채 누적 → 어느 시점부터 신규 환경에서 깡통 부팅 불가 |
| **원인** | Prisma는 마이그레이션 검증 시 **빈 Shadow DB → 순차 모든 마이그레이션 재실행**으로 정합 확인. 다음 패턴이 박혀있으면 재실행 불가: (a) RENAME (`audit.audit_events` → `public.spx_audit_events`) — 옛 schema 부재 (b) 스키마 간 객체 이동 — 출발 schema 부재 (c) 운영-신규 환경 출발 상태 차이를 한 마이그레이션에 담음 — 운영 가정 코드가 신규 환경에 적용 시 에러 (d) MView/View가 `IF NOT EXISTS` 없는 `CREATE` 사용 — 재실행 시 중복 |
| **영향** | (a) 신규 합류자 환경 셋업 시 `docker compose up`만으로 부팅 불가 — 매번 수동 우회 안내 필요 (b) CI에 마이그레이션 검증 도입 불가 — Shadow DB 재생 의존이라 깨짐 (c) 부채가 누적될수록 우회 비용 기하급수 증가 (d) Prisma 약점 영역(RENAME / 스키마 이동)에 무리하게 박은 학습 — Prisma는 이런 패턴 핸들링 약함 |
| **방어** | **첫 마이그레이션부터 idempotent 의무**: (a) 모든 DDL을 `CREATE ... IF NOT EXISTS` / `CREATE OR REPLACE` / `DROP ... IF EXISTS` 패턴 (b) RENAME/스키마 이동은 `DO $$ BEGIN IF EXISTS(...) THEN EXECUTE 'RENAME ...' END IF; END $$` 조건부 블록 (c) 운영-신규 환경 차이는 `IF EXISTS (SELECT 1 FROM information_schema.tables WHERE ...)` 조건부 데이터 이전 (d) **entrypoint baseline 분기**: `_prisma_migrations` 부재 시 자동 적용 (5/18 갈래 1+2 합성 솔루션). 마이그레이션 표준은 `hdd/delegation-standard.md § 마이그레이션 본문 작성 표준` 참조 |
| **테스트** | (a) `docker compose down -v && docker compose up -d` 깡통 부팅에서 모든 마이그레이션 정상 적용 (b) 운영 가정 시뮬레이션 — 옛 상태 dump 위에 마이그레이션 적용 시 정상 (c) `prisma migrate dev --create-only` 실행 시 Shadow DB 재생 에러 없음 |
| **관련** | [[4. 지식노트/Prisma migrate - Shadow DB와 entrypoint 의존성 부채]], `dify-audit/prisma/audit/migrations/20260515000000_rename_audit_to_spx_and_move_to_public/`, 5/18 갈래 1 + 갈래 2 합성 (SESSION_HISTORY 2026-05-18) |

### H-INFRA-01b: Self-heal 부분 실행이 mart를 더 손상시킴 (CASCADE 연쇄 제거)

| 항목 | 내용 |
|------|------|
| **상황** | 194 환경에서 `spx_mv_model_tokens_daily` 누락 + `spx_mv_audit_enriched` 중복(2개) + zombie `spx_v_audit_enriched` 발견 (2026-06-10). entrypoint self-heal이 반복 실행되면서 상태가 점점 악화 |
| **원인** | self-heal 블록이 `-v ON_ERROR_STOP=1` + `set -e`로 구성 → 20260519 마이그레이션의 `DROP MATERIALIZED VIEW ... CASCADE`가 Layer2(kpi_calls, model_tokens) 먼저 제거 → 뒤 단계에서 enriched 재생성 에러 시 HALT → model_tokens 재생성(20260610) 전에 멈춤. 재시작마다 반복되어 중복/zombie 누적 |
| **영향** | (a) model_tokens_daily 누락 → 모델 차트 빈 화면 (b) enriched 중복/zombie → refresh chain 실패(`XX001` 스토리지 손상 유발 가능) (c) 재시작마다 악화 — self-heal이 치유가 아니라 추가 손상 (d) `set -e`로 entrypoint 자체가 중단 → 컨테이너 서비스 불능 |
| **방어** | (a) **self-heal 블록에서 `ON_ERROR_STOP=1` 제거** — 개별 마이그레이션 에러가 후속 마이그레이션을 차단하면 안 됨 (b) 각 마이그레이션을 `if ! psql ...; then echo WARNING; fi` 패턴으로 에러 카운트 + 계속 진행 (c) B안(model_tokens)은 enriched 독립 의존이라 enriched 에러와 무관하게 복구 가능 — 이 디커플링이 방어 역할 (d) **2026-06-10 적용**: entrypoint.sh self-heal 블록 경화 완료 |
| **테스트** | (a) 로컬 DB에서 self-heal 리스트 2회 연속 실행 → enriched 1개 / zombie 0 / kpi_calls·model_tokens 정상 (b) 에러 있는 마이그레이션 끼워 넣어도 후속 마이그레이션 실행되는지 |
| **관련** | H-INFRA-01(Shadow DB 부채), `dify-audit/scripts/entrypoint.sh` self-heal 블록, B안 마트 재작성(20260610) |

### H-INFRA-02: Docker compose 옛 mount 정의 보존 — restart로는 새 정의 미반영

| 항목 | 내용 |
|------|------|
| **상황** | `docker-compose.yaml`의 volume 경로 변경(예: keycloak realm 파일 mount 경로 수정) 후 `docker compose restart <service>`로는 새 정의 반영 안 됨. 옛 경로가 호스트에 없으면 Docker Desktop이 빈 디렉토리 자동 생성 → "디렉토리를 파일에 마운트" 에러로 컨테이너 startup 실패 |
| **원인** | `restart` 는 컨테이너 프로세스만 재시작, 컨테이너 정의(volume mount 등)는 옛것 유지. 새 compose 정의를 반영하려면 컨테이너 자체를 destroy + recreate 필요. 또한 Windows Docker Desktop이 mount 경로 부재 시 빈 디렉토리를 호스트에 자동 생성하는 동작 → 파일 mount 의도였는데 디렉토리가 잡혀 mount 에러 |
| **영향** | (a) "compose 고쳤는데 왜 안 됨?" 디버깅에 시간 소요 (b) 호스트에 빈 디렉토리 잔존 → 다음 compose up 시도에도 같은 에러 반복 (c) 5/19 keycloak realm 파일 mount 사고: 옛 경로 mount 잔존 + 호스트 빈 디렉토리 자동 생성 콤보로 컨테이너 무한 restart 루프 |
| **방어** | (a) **compose 변경 시 `docker compose up -d --force-recreate <service>` 의무** — `restart` 단독 금지 (b) 빈 디렉토리 잔존 의심 시 `Get-ChildItem <mount경로> -Directory` 또는 `ls -la` 검증 후 `Remove-Item -Recurse <경로>` 수동 정리 (c) compose `volumes:` 항목에 `:ro` 또는 명시적 `bind` type 사용해 의도 명확화 |
| **테스트** | (a) compose volume 경로 변경 후 `force-recreate` 적용 → `docker inspect <container> --format '{{json .Mounts}}'` 출력에 새 경로 반영 (b) 호스트에 의도하지 않은 빈 디렉토리 0건 |
| **관련** | [[1. Daily/2026-05-19.md]] keycloak 컨테이너 사고, [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]], H-ENV-02 (api/worker build 지시 부재) |

### H-INFRA-04: Keycloak admin API 토큰 만료 (대량 처리 중 401)

| 항목 | 내용 |
|------|------|
| **상황** | `scripts/sync_dept_keycloak.sh` / `sync_mock_users_keycloak.py` 대량 처리 중 Keycloak admin API access token이 5분 만에 만료되어 50건 이상 처리 도중 401 반환. 5/22 부서 10개 + 사용자 150명 동기화 작업 중 발견 |
| **원인** | Keycloak `admin-cli` client의 access token TTL이 기본 5분(realm 설정에 따라 다름). 부서/사용자 50건 이상 처리 시 한 사이클 안에 만료. Token refresh 미구현 시 중간 fail |
| **영향** | (a) 동기화 50건 처리 도중 401 → 중간 실패 (b) 재실행 시 부분 반영 상태에서 멱등성 검증 필요 — 일부 그룹/사용자는 이미 생성, 일부는 미생성 (c) 토큰 만료 시점 로그 추적 안 되면 "왜 갑자기 401?" 디버깅 시간 소요 |
| **방어** | (a) **50건마다 또는 4분 경과마다 토큰 재발급** (refresh_token 또는 password grant 재요청) — 5/22 채택 패턴 (b) 또는 client TTL을 30분으로 늘림 (운영 결정 필요 — Keycloak realm `accessTokenLifespan` 설정) (c) 스크립트 멱등 보장: `NOT EXISTS` 체크 + 부분 반영 상태에서 재실행 안전 (d) 토큰 발급/만료 시각 로그 명시 |
| **테스트** | (a) 60건 부서 동기화 dry-run 후 토큰 재발급 횟수 로그 출력 (1회 이상) (b) 401 발생 시 자동 재발급 + 재시도 통과 (c) 부분 반영 상태에서 재실행 시 추가 변경 0건 |
| **관련** | SESSION_HISTORY 2026-05-22 § 트러블슈팅, H-ENV-04 (Git Bash 멀티바이트 — 같은 5/22 sync 작업 자매 함정), [[4. 지식노트/Keycloak admin API 토큰 만료 대응]] (작성 예정) |

### H-INFRA-03: dist 빌드 캐시 함정 — restart로는 새 src 반영 안 됨

| 항목 | 내용 |
|------|------|
| **상황** | TypeScript 등 컴파일 결과물(`dist/`)이 컨테이너 이미지에 박혀있는 서비스(dify-audit collector 등). src 변경(특히 raw query 박힌 collectors) 후 `docker compose restart`만으론 옛 dist 그대로 — 빌드 안 거치니 src 변경 무반영. 동료에게 "src 갱신했으니 restart 해주세요" 안내 시 발생 |
| **원인** | 컨테이너 이미지 빌드 시 `npm run build`로 `src/` → `dist/` 컴파일 후 image layer에 박힘. 컨테이너 부팅은 `dist/`를 실행. `restart`는 같은 이미지로 프로세스만 재시작 → dist 동일. **build + image rebuild + recreate** 3단계 필요 |
| **영향** | (a) 동료가 `git pull → restart`만 하면 옛 동작 그대로 — "왜 안 됨?" 신고 (b) 특히 raw SQL query가 dist에 박혀있는 경우 (예: 5/19 dify-audit collector의 `accounts` → `spx_accounts` rename 무반영 → 옛 테이블명으로 쿼리 → table not found 에러) (c) 디버깅 시 src와 컨테이너 실 동작 괴리로 혼란 |
| **방어** | (a) **동료 안내 표준 — `git pull → docker compose build --no-cache <service> → docker compose up -d --force-recreate <service>` 3종 세트**. `restart` 단독 금지 명시. ⚠️ **`--no-cache` 필수 (2026-06-10 실측)**: 일반 `build`는 빌드 캐시가 구 소스로 컴파일해 변경 미반영 → src 바꿨는데도 옛 dist. (b) src 변경 동반 PR 머지 시 슬랙 안내에 build 명령 박기 (c) entrypoint script에 `npm run build` 박기 검토(이미지 빌드 시간 vs 매 부팅 시간 trade-off, 일반적으로 비추) (d) CI 이미지 빌드 자동화 + 동료 환경은 pull만으로 새 이미지 받기 |
| **테스트** | (a) src 변경 후 `restart`만 했을 때 옛 동작 재현 (b) `build + up --force-recreate` 후 새 동작 반영 확인 — `docker exec <container> cat /app/dist/<file>.js | grep <변경 토큰>` (c) 동료 환경에서 `git pull + build + up -d --force-recreate` 3종 세트로 정상 적용 검증 |
| **관련** | [[1. Daily/2026-05-19.md]] dify-audit collector raw query 옛 빌드 잔존 사고, H-INFRA-02 (compose mount drift), [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]] |

---

## 문서/정합 결함 (2026-05-19 신설 카테고리)

### H-DOC-01: design.md 본 갱신 ↔ Prisma 마이그레이션 후속 작업 분리 (drift 누적 패턴)

| 항목 | 내용 |
|------|------|
| **상황** | `hdd/design.md § 2.5.4` 본 갱신만 박히고 Prisma 마이그레이션 파일 추가 누락 (또는 그 반대). 본 갱신 시점엔 정합 보이지만 운영 적용 시 실제 DB DDL과 design.md 어긋남 → "design.md 보고 작성한 쿼리가 왜 안 됨?" 신고 → drift 발견 → 정정 |
| **원인** | (a) 설계 변경은 자연스러운 "본 먼저 갱신 → 코드/마이그레이션 차차"의 흐름인데 후속 작업이 끊김 (b) 본 갱신 PR과 마이그레이션 PR이 별도라 머지 시점 차이로 일시적 drift 정상화 (c) **본 갱신만으로 "끝났다" 자기 검증 거짓** — 마이그레이션 적용 + 검증 안 한 채 종료 보고 (d) 5/15 audit rename / 5/18 drift 1·2차 모두 같은 가족 결함 — 동일 패턴 누적 |
| **영향** | (a) design.md를 진실원으로 신뢰한 후속 작업이 모두 어긋남 (b) 동료/AI 위임 시 design.md만 보고 작성 → 실제 DB와 안 맞음 → 재작업 (c) drift 발견까지의 잠복 기간이 길면 누적 drift 풀기 비용 폭증 (d) "design.md 신뢰도" 자체 흔들림 — 같이 일하는 사람이 본 갱신 안 보고 매번 `\d` 직접 보는 회피 패턴 형성 |
| **방어** | (a) **본 갱신 PR 시 마이그레이션 파일 동시 commit 의무** — 한 PR에 design.md + migrations/* 같이 박힘 (b) **drift 게이트(quality-criteria § 2 백엔드)로 사후 감지** — PR 직전 design.md DDL vs 실제 DB `\d+` 출력 1:1 정합 검증 (c) 위임 프롬프트에 "drift 발견 시 작업 중단 + 사용자 보고" 가드 박기 (`hdd/delegation-standard.md`) (d) 본 갱신만 박힌 PR은 자동 차단(미래) — pre-commit hook이 design.md DDL 코드 블록 변경 감지 시 마이그레이션 파일 동반 변경 확인 |
| **테스트** | (a) `scripts/check-mart-drift.ps1`(작성 예정) 또는 수동 `psql \d+` vs design.md grep diff 0건 (b) 본 갱신 PR에 마이그레이션 파일 변경 동반 grep — 동반 없으면 PR 코멘트로 차단 (c) drift 발견 시 작업 중단 의무 시뮬레이션 — 가상 PR에 drift 박은 후 위임 → 작업 중단 + 사용자 보고 흐름 작동 확인 |
| **관련** | [[3. 프로젝트/spx-agent/hdd/quality-criteria.md]] § 2 백엔드 drift 게이트, `hdd/delegation-standard.md`, 5/15 audit rename 사고 / 5/18 drift 1·2차 (SESSION_HISTORY) |

---

## Harness Candidates (검증 대기)

> **방어용 패턴이 될 가능성이 있으나 정식 등록 전 단계**.
> 5/4 일지의 "가설 결함을 catalog에 박지 마라" 원칙: 1차 발생 시점엔 가설로만 표시, 추가 사례 발생 시 정식 H-XX-YY ID 부여 + 위 카탈로그로 승격.
>
> **승격 트리거**: 1건 추가 발생 (재발 입증) 또는 소스 분석으로 확정.

### CAND-husky-activation-drift

| 항목 | 내용 |
|------|------|
| **가설** | husky의 `pnpm install` prepare script 활성화 시점이 개발자별로 달라 lint 사후 위반 발견 발생. 한 개발자가 hook 미활성 환경에서 commit한 코드가 그대로 master 진입 → 이후 다른 개발자가 hook 활성화된 환경에서 commit 시도 → 그 사람 commit이 남의 위반으로 막힘 |
| **현 사례** | 2026-05-06 권수현님 `feat/rbac` 브랜치 keycloak.py 외 2파일에 ruff TRY400(4)+E501(3) 위반. 본인 환경에서 commit 시도 시 발견 → 권수현님 본인 환경에선 hook 미활성으로 추정 |
| **검증 필요** | (a) 추가 1건 더 같은 패턴 발생 시 정식 H-ENV-04 승격 (b) 또는 `.husky` 활성화 확인 명령(`git config --get core.hooksPath`)을 셋업 가이드에 박은 후 신규 개발자에서 같은 함정 발견되면 즉시 승격 |
| **임시 방어** | (a) 셋업 절차에 `pnpm install` 후 `git config --get core.hooksPath` 검증 단계 명시 (`architecture.md § 9.2`) (b) 다른 사람 영역 코드 위반 발견 시 슬랙 알림 + `--no-verify` 우회 (본인 영역만 commit) (c) **CI lint job이 유일한 강제 게이트** — 회사 차원에서 CI ruff/eslint 강제 권장 |
| **관련** | [[4. 지식노트/husky - .husky 폴더 패턴과 install 시점]], [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (2026-05-06 변경 이력) |

### CAND-git-checkout-with-running-container

| 항목 | 내용 |
|------|------|
| **가설** | 컨테이너가 코드 디렉토리를 bind mount로 잡은 채 `git checkout`/브랜치 전환을 하면, 새 브랜치에 없는 파일을 git이 삭제하지 못하고 **빈 디렉토리 형태로 워킹트리에 잔존**. 다음 컨테이너 부팅 시 호스트의 깨진 형태(빈 디렉토리)를 그대로 마운트 → Python `.py` import 불가 → flask_migrate가 `Can't locate revision identified by 'XXX'` 무한 반복 → API/worker 컨테이너 health: starting 무한 재시작 → localhost 무한 로딩 |
| **현 사례** | 2026-05-07 오전. 어제 5/6 권대리님 `feat/rbac` 브랜치 마이그레이션 적용 후 `KAN-29-...` 브랜치로 돌아온 상태에서 발생. `api/migrations/versions/2026_04_27_1200-a1b2c3d4e5f6_add_rbac_tables.py` + `2026_04_28_1000-c1d2e3f4a5b6_make_owner_department_id_nullable.py` 2개 파일이 호스트에서 `Mode: d-----` (빈 디렉토리)로 깨짐. git ls-tree에는 정상 blob (`100644`)으로 존재. DB의 `alembic_version`엔 `c1d2e3f4a5b6` 박혀있어 코드/DB 미스매치로 부팅 거부 |
| **검증 필요** | (a) 추가 1건 발생 시 정식 H-ENV-04 승격 (b) 또는 동일 메커니즘이 다른 파일 유형(.json/.yaml/.ts)에서도 재현되면 즉시 승격 (c) Linux/macOS 호스트에서도 발생하는지 확인 — Windows의 file-in-use 동작이 핵심 트리거인지 검증 |
| **임시 방어** | (a) 권장: 브랜치 전환 전 `docker compose stop api worker worker_beat` (코드 mount 잡은 컨테이너 멈춤) (b) 사후 검증: `Get-ChildItem api\migrations\versions -Directory \| Where-Object Name -Match '\.py$'` 결과 0건 (c) 발견 시 복구: 빈 디렉토리 삭제 → `git checkout origin/<src-br> -- <path>`로 복원 (현 브랜치 HEAD에 없으면 source 브랜치 명시) → 컨테이너 재시작 (d) 검증: `docker exec <api> sh -c "cd /app/api && flask db heads"` head revision 출력 |
| **테스트** | (a) `docker exec <api> ls -la /app/api/migrations/versions/<file>.py` → `-rwxrwxrwx ... <size>` (디렉토리 X) (b) API 컨테이너 status `Up X minutes (healthy)` (c) `Invoke-WebRequest http://localhost` Status 200 |
| **관련** | [[1. Daily/2026-05-07.md]] 1차 발생, H-ENV-01/02 (docker bind mount 함정 계열), [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]] |

---

## 카탈로그 요약

| ID | 영역 | 결함 패턴 | 심각도 |
|----|------|---------|:------:|
| H-DASH-01 | 데이터 집계 | ADVANCED_CHAT 토큰 이중 카운트 | 상 |
| H-DASH-02 | 데이터 집계 | WORKFLOW/챗플로우 모델별 분리 불가 (모델은 노드에만) | 중 |
| H-DASH-03 | 데이터 집계 | 디버깅 실행 데이터 혼입 | 상 |
| H-DASH-04 | 데이터 집계 | 레거시 앱 부서 미배정 누락 | 중 |
| H-DASH-05 | 기간·데이터 | Celery Beat 삭제로 이전 기간 없음 | 중 |
| H-DASH-06 | 기간·데이터 | 비동기 저장 지연으로 최신 누락 | 하 |
| H-DASH-07 | 계산·표시 | 증감률 분모 0 → NaN | 상 |
| H-DASH-08 | 계산·표시 | 활성 사용자 NULL 포함 | 중 |
| H-DASH-09 | 계산·표시 | 로컬 모델 표시 구분 | 하 |
| H-DASH-10 | 성능 | 부서별 활동 다단 JOIN 느림 | 중 |
| H-DASH-11 | 성능 | JSON 파싱 성능 | 중 |
| H-DASH-12 | 성능 | CSV 대량 데이터 OOM | 중 |
| H-DASH-13 | 연동·충돌 | RBAC 스키마 변경 영향 | 상 |
| H-DASH-14 | 연동·충돌 | users vs accounts 불일치 | 상 |
| H-DASH-15 | 연동·충돌 | 설정 모달 사이드바 충돌 | 중 |
| H-DASH-16 | Spec 품질 | spec과 화면 설계 이미지/PDF 어긋남 (자기 검토 함정) | 상 |
| H-DASH-17 | Spec 품질 | 외부 의존 명세를 추정으로 박는 함정 (관찰 부족) | 상 |
| H-DASH-18 | 데이터 집계 | Dify 예외 타입 DB 미보존 → messages.error 분류는 ILIKE만 | 상 |
| H-ENV-01  | 빌드·환경 | Linux .venv lib64 symlink로 BuildKit 실패 (재발성) | 상 |
| H-ENV-02  | 빌드·환경 | api/worker 서비스에 build 지시 없어 호스트 코드 미반영 | 상 |
| H-ENV-03  | 빌드·환경 | 호스트 Node 버전 drift로 vitest/ESLint hook 부팅 불가 (재발성) | 상 |
| H-MART-01 | 마트·데이터 | enriched view 비-UUID actor 캐스트 함정 | 상 |
| H-INFRA-01 | 인프라 | Shadow DB 부채 — Prisma migrate dev 재생 불가 누적 | 상 |
| H-INFRA-01b | 인프라 | Self-heal 부분 실행이 CASCADE로 mart 추가 손상 (6/10 194 사고) | 상 |
| H-INFRA-02 | 인프라 | Docker compose 옛 mount 정의 보존 — restart로는 새 정의 미반영 | 상 |
| H-INFRA-03 | 인프라 | dist 빌드 캐시 함정 — restart로는 새 src 반영 안 됨 | 상 |
| H-DOC-01  | 문서·정합 | design.md 본 갱신 ↔ Prisma 마이그레이션 후속 작업 분리 (drift 누적) | 상 |
| CAND-husky-activation-drift | 빌드·환경 | husky 활성화 시점 차이로 lint 사후 위반 발견 (1건, 검증 대기) | 중 |

## 관련 노트

- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 하네스 설계.md|하네스 설계]]
- [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]]
- [[4. 지식노트/Dify - workflow_node_executions에서 모델별 토큰 추출.md]]
- [[4. 지식노트/Dify - Celery Beat 정기 백그라운드 작업.md]]
- [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
