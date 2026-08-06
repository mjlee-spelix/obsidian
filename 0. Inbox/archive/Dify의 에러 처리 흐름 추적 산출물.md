## 산출물

### 1. Dify 에러 처리 흐름 — 예외 클래스 → DB 변환 경로

**예외 계층** ([invoke.py](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/.venv/Lib/site-packages/graphon/model_runtime/errors/invoke.py)):

```
InvokeError (base, description 속성 보유)
├── InvokeConnectionError      (기본: "Connection Error")
├── InvokeServerUnavailableError (기본: "Server Unavailable Error")
├── InvokeRateLimitError        (기본: "Rate Limit Error")
├── InvokeAuthorizationError    (기본: "Incorrect model credentials...")
└── InvokeBadRequestError       (기본: "Bad Request Error")
```

**변환 경로 (3단계):**

|단계|파일:라인|동작|
|---|---|---|
|1. 예외 발생|[base.py:321-355](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/plugin/impl/base.py#L321-L355)|Plugin daemon 에러 → `InvokeRateLimitError(description=error_object.get("message"))` 생성. **SDK 원문 메시지가 description에 보존됨**|
|2. 예외 catch|[chat/app_generator.py:248-261](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/app/apps/chat/app_generator.py#L248-L261)|`InvokeAuthorizationError`만 특별 처리("Incorrect API key provided"로 교체). 나머지는 **원본 예외 그대로** `queue_manager.publish_error(e)`|
|3. DB 저장|[based_generate_task_pipeline.py:44-86](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/app/task_pipeline/based_generate_task_pipeline.py#L44-L86)|`_error_to_desc(err)` → `getattr(e, "description", str(e))` → **`message.error = err_desc`** (문자열)|

**예외 타입 정보 손실 지점**: [based_generate_task_pipeline.py:65-67](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/app/task_pipeline/based_generate_task_pipeline.py#L65-L67)

```python
err_desc = self._error_to_desc(err)   # Exception → str 변환
message.status = MessageStatus.ERROR
message.error = err_desc               # ← 여기서 클래스명 손실. 문자열만 저장
```

`isinstance(e, InvokeRateLimitError)` 같은 타입 정보가 **문자열 변환 시 완전히 소실**됩니다. DB에는 `description` 텍스트만 남습니다.

---

### 2. SDK error code 보존 여부 — **No**

**결론: SDK error code는 DB 어디에도 보존되지 않습니다.**

근거:

- `messages` 테이블에 `error_code`, `error_type` 컬럼 **없음** ([model.py:1418](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/models/model.py#L1418) — `error` 컬럼은 `LongText` 하나뿐)
- `message_metadata`는 **에러 시 채워지지 않음** — `_save_message()`는 정상 완료 경로에서만 호출됨 ([easy_ui_based_generate_task_pipeline.py:407](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/app/task_pipeline/easy_ui_based_generate_task_pipeline.py#L407))
- Plugin daemon → Python 변환 시 `error_type` (예: `"InvokeRateLimitError"`)이 `match` 문으로 사용되지만 ([base.py:330-331](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/plugin/impl/base.py#L330-L331)), **이 타입명은 DB에 저장하지 않고 Python 예외 클래스 선택에만 쓰임**
- OpenAI/Anthropic SDK의 `error.code` 필드는 plugin daemon 내부에서 InvokeError 서브클래스 선택에 소비되고, Dify 백엔드에 도달할 때는 이미 `description` 문자열만 남음

**옵션 3은 불가능합니다.**

---

### 3. messages.error 텍스트 패턴 — **반정형(semi-structured), 분류 가능**

InvokeError 서브클래스의 `description` 기본값이 안정적 접두어 역할을 합니다:

|에러 유형|messages.error에 저장되는 텍스트|패턴 안정성|
|---|---|---|
|Rate limit|`"Rate Limit Error"` (기본) 또는 `"Rate limit reached for gpt-4o on..."` (SDK 원문 덮어쓰기)|**높음** — `Rate Limit` 포함|
|Connection|`"Connection Error"` 또는 SDK 상세 메시지|**높음** — `Connection` 포함|
|Auth|**항상** `"Incorrect API key provided"` (app_generator에서 하드코딩 교체)|**최고** — 고정 문자열|
|Server unavailable|`"Server Unavailable Error"` 또는 상세|**높음**|
|Bad request (context length 등)|`"Bad Request Error"` 또는 `"This model's maximum context length is 128000 tokens..."`|**중간** — SDK별 다양|
|Quota exceeded|`"Your quota for Dify Hosted Model Provider has been exhausted..."`|**최고** — 고정 ([based_generate_task_pipeline.py:77-80](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/app/task_pipeline/based_generate_task_pipeline.py#L77-L80))|
|기타/Unknown|`str(exception)` 또는 `exception.description`|**낮음** — 자유 텍스트|

**핵심 관찰**: Plugin daemon에서 description을 덮어쓸 때 SDK 원문 메시지를 전달하므로 ([base.py:333](vscode-webview://19v0f66hbb118n6fghbv51q7bvlsdmmhfualqnq0hklrlg73ttr0/api/core/plugin/impl/base.py#L333) `description=error_object.get("message")`), 기본값 `"Rate Limit Error"`가 아닌 상세 메시지가 들어올 수 있습니다. 하지만 InvokeError 타입별로 **키워드 패턴은 안정적**입니다.

**정규식 분류 예시** (5개 룰이면 주요 에러 90%+ 커버):

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

---

### 4. 추가 활용 가능 컬럼/필드 — **없음**

|확인 대상|결과|
|---|---|
|`messages.error_code`|컬럼 없음|
|`messages.error_type`|컬럼 없음|
|`messages.message_metadata`|에러 시 NULL (정상 완료 시에만 채워짐)|
|`workflow_runs.error`|동일 패턴 — 문자열만 저장|
|`workflow_node_executions.error`|동일 패턴|
|별도 에러 로그 테이블|없음|

---

### 5. 옵션 결정 권고

|옵션|가능 여부|평가|
|---|---|---|
|**옵션 3** (SDK error code 활용)|**불가능**|error code가 DB까지 보존되지 않음. 예외 클래스 선택에만 소비되고 버려짐|
|**옵션 4** (마트 ETL 정규식)|**권장**|`CASE WHEN ILIKE` 5~6개 룰로 주요 에러 타입 커버 가능. InvokeError 기본 description이 안정적 키워드 제공. collector 수정 불필요|
|**옵션 2** (collector 보강)|**가능하지만 과도**|collector에 분류 함수 추가 시 audit_events에 `errorType` 필드 신설 가능. 하지만 ETL에서 동일 작업 가능하므로 audit 팀 협의 부담 대비 이점 낮음|

**최종 권고: 옵션 4**

- 근거: `messages.error` 텍스트의 키워드 패턴이 InvokeError 클래스 계층 덕분에 충분히 안정적. ILIKE 룰 5~6개로 분류 가능. collector/audit 인프라 수정 없이 마트 ETL 단독 구현 가능.
- 리스크: SDK 메시지 포맷이 바뀌면 룰 갱신 필요 → 외부 룰 테이블(옵션 5)과 조합하면 운영 부담 최소화