---
tags: [프로젝트, dify, AI-Agent, references]
type: references
date: 2026-05-12
last_updated: 2026-05-12
source: VSCode Claude 조사 산출물 (2026-05-12)
purpose: top-error-types 차트 구현 + 옵션 D(정규식 ETL) 분류 룰 카탈로그 근거
related_defects: H-DASH-18
---

# Dify 에러 처리 흐름

> Dify에서 LLM/API 호출 에러가 발생했을 때 예외 객체가 어떻게 변환되어 DB(`messages.error`)에 저장되는지의 전체 경로.
> top-error-types 차트(⑱ 부서별 주요 에러 유형) 구현의 분류 룰 근거.

## 1. 예외 계층 — `InvokeError`

위치: `api/.venv/Lib/site-packages/graphon/model_runtime/errors/invoke.py`

```
InvokeError (base, description 속성 보유)
├── InvokeConnectionError       (기본: "Connection Error")
├── InvokeServerUnavailableError (기본: "Server Unavailable Error")
├── InvokeRateLimitError         (기본: "Rate Limit Error")
├── InvokeAuthorizationError     (기본: "Incorrect model credentials...")
└── InvokeBadRequestError        (기본: "Bad Request Error")
```

## 2. 변환 경로 — 3단계

| 단계 | 파일:라인 | 동작 |
|---|---|---|
| 1. 예외 발생 | `api/core/plugin/impl/base.py:321-355` | Plugin daemon 에러 → `InvokeRateLimitError(description=error_object.get("message"))` 생성. **SDK 원문 메시지가 description에 보존됨** |
| 2. 예외 catch | `api/core/app/apps/chat/app_generator.py:248-261` | `InvokeAuthorizationError`만 특별 처리("Incorrect API key provided"로 교체). 나머지는 **원본 예외 그대로** `queue_manager.publish_error(e)` |
| 3. DB 저장 | `api/core/app/task_pipeline/based_generate_task_pipeline.py:44-86` | `_error_to_desc(err)` → `getattr(e, "description", str(e))` → **`messages.error = err_desc`** (문자열) |

## 3. 예외 타입 정보 손실 지점 ⭐

위치: `api/core/app/task_pipeline/based_generate_task_pipeline.py:65-67`

```python
err_desc = self._error_to_desc(err)   # Exception → str 변환
message.status = MessageStatus.ERROR
message.error = err_desc               # ← 여기서 클래스명 손실. 문자열만 저장
```

`isinstance(e, InvokeRateLimitError)` 같은 타입 정보가 **문자열 변환 시 완전히 소실**. DB에는 `description` 텍스트만 남는다.

→ **함의**: 향후 누군가 `messages.error`로 에러 분류 기능 추가할 때 `isinstance` 흐름 기대 금지. 텍스트 매칭(ILIKE)만이 유일한 길. ([[3. 프로젝트/spx-agent/hdd/defect-catalog.md#H-DASH-18]])

## 4. SDK error code 보존 여부 — **No**

**결론: SDK error code는 DB 어디에도 보존되지 않는다.**

근거:

- `messages` 테이블에 `error_code`, `error_type` 컬럼 **없음** (`api/models/model.py:1418` — `error` 컬럼은 `LongText` 하나뿐)
- `message_metadata`는 **에러 시 채워지지 않음** — `_save_message()`는 정상 완료 경로에서만 호출됨 (`api/core/app/task_pipeline/easy_ui_based_generate_task_pipeline.py:407`)
- Plugin daemon → Python 변환 시 `error_type` (예: `"InvokeRateLimitError"`)이 `match` 문으로 사용되지만 (`api/core/plugin/impl/base.py:330-331`), **이 타입명은 DB에 저장하지 않고 Python 예외 클래스 선택에만 쓰임**
- OpenAI/Anthropic SDK의 `error.code` 필드는 plugin daemon 내부에서 InvokeError 서브클래스 선택에 소비되고, Dify 백엔드에 도달할 때는 이미 `description` 문자열만 남음

→ **옵션 3(SDK error code 활용)은 영구 불가**. collector 보강(옵션 2) 또는 Dify upstream 변경 없이는 회피 경로 없음.

## 5. `messages.error` 텍스트 패턴 — 반정형(semi-structured)

InvokeError 서브클래스의 `description` 기본값이 안정적 접두어 역할을 한다.

| 에러 유형 | `messages.error` 텍스트 | 패턴 안정성 |
|---|---|---|
| Rate limit | `"Rate Limit Error"` (기본) 또는 `"Rate limit reached for gpt-4o on..."` (SDK 원문 덮어쓰기) | **높음** — `Rate Limit` 포함 |
| Connection | `"Connection Error"` 또는 SDK 상세 메시지 | **높음** — `Connection` 포함 |
| Auth | **항상** `"Incorrect API key provided"` (app_generator에서 하드코딩 교체) | **최고** — 고정 문자열 |
| Server unavailable | `"Server Unavailable Error"` 또는 상세 | **높음** |
| Bad request (context length 등) | `"Bad Request Error"` 또는 `"This model's maximum context length is 128000 tokens..."` | **중간** — SDK별 다양 |
| Quota exceeded | `"Your quota for Dify Hosted Model Provider has been exhausted..."` | **최고** — 고정 (`api/core/app/task_pipeline/based_generate_task_pipeline.py:77-80`) |
| 기타/Unknown | `str(exception)` 또는 `exception.description` | **낮음** — 자유 텍스트 |

**핵심 관찰**: Plugin daemon에서 description을 덮어쓸 때 SDK 원문 메시지를 전달하므로 (`api/core/plugin/impl/base.py:333` — `description=error_object.get("message")`), 기본값 `"Rate Limit Error"`가 아닌 상세 메시지가 들어올 수 있다. 하지만 InvokeError 타입별로 **키워드 패턴은 안정적**이다.

## 6. 분류 룰 카탈로그 — 옵션 D(정규식 ETL)용

5~6개 룰로 주요 에러 90%+ 커버:

```sql
CASE
  WHEN error ILIKE '%rate limit%'          THEN 'rate_limit'
  WHEN error ILIKE '%incorrect api key%'   THEN 'auth_error'
  WHEN error ILIKE '%quota%exhausted%'     THEN 'quota_exceeded'
  WHEN error ILIKE '%connection error%'    THEN 'connection_error'
  WHEN error ILIKE '%context length%'      THEN 'context_length'
  WHEN error ILIKE '%server unavailable%'  THEN 'server_unavailable'
  ELSE 'unknown'
END
```

룰 갱신 빈도 임계 도달 시 → **옵션 5(외부 룰 테이블)** 도입 검토 (룰을 DB 또는 설정 파일로 분리, ETL 코드 수정 없이 운영). 도입 시점은 [[3. 프로젝트/spx-agent/hdd/design.md]] § 10 미해결 결정 참조.

## 7. 추가 활용 가능 컬럼/필드 — 없음

| 확인 대상 | 결과 |
|---|---|
| `messages.error_code` | 컬럼 없음 |
| `messages.error_type` | 컬럼 없음 |
| `messages.message_metadata` | 에러 시 NULL (정상 완료 시에만 채워짐) |
| `workflow_runs.error` | 동일 패턴 — 문자열만 저장 |
| `workflow_node_executions.error` | 동일 패턴 |
| 별도 에러 로그 테이블 | 없음 |

## 8. 옵션 결정 요약

| 옵션 | 가능 여부 | 평가 |
|---|---|---|
| **옵션 C** (audit.action 활용) | ❌ 불가능 | 2026-05-11 확정 — audit collector 분류 함수 0 |
| **옵션 3** (SDK error code 활용) | ❌ 불가능 | DB 미보존. plugin daemon 내부에서 소비되고 버려짐 |
| **옵션 D** (마트 ETL 정규식) | ✅ **권장·채택** | ILIKE 5~6 룰로 주요 에러 90%+ 커버. collector/audit 수정 불필요 |
| **옵션 E** (절충: 호출=OLTP / 보안=audit) | ⏸ PM 응답 대기 | top-error-types 차트 범위 확인 필요 (호출 에러만 / 보안 이벤트 포함 / 둘 다) |
| **옵션 2** (collector 보강) | 가능하지만 과도 | audit_events에 `errorType` 필드 신설 가능. 단 ETL에서 동일 작업 가능하므로 audit 팀 협의 부담 대비 이점 낮음 |
| **옵션 5** (외부 룰 테이블) | 보류 | 룰 갱신 빈도 임계 도달 후 도입. 옵션 D 위에 얹는 형태 |

## 관련

- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md#H-DASH-18]] — 본 흐름에서 도출된 결함 패턴
- [[3. 프로젝트/spx-agent/hdd/design.md]] § 2 — OLTP 3-fact 마트 설계 (옵션 D 룰 카탈로그 박는 위치)
- [[3. 프로젝트/spx-agent/hdd/design.md]] § 10 — 미해결 결정 (옵션 E/5)
- [[3. 프로젝트/spx-agent/references/dify-db-schema.md]] — Dify 테이블 스키마
- [[1. Daily/2026-05-12.md]] — 옵션 결정 일지
- [[0. Inbox/archive/Dify의 에러 처리 흐름 추적 산출물.md]] — 원본 산출물 (이관 후 archive)
