---
tags: [Python, 개발, 백엔드, Pydantic]
date: 2026-05-04
---
# Pydantic - Python 데이터 검증 라이브러리

## 핵심

> Python에서 **"데이터 형태"를 클래스로 선언하고 자동 검증·직렬화**해주는 라이브러리. Python 타입 힌트를 런타임 검증으로 활용.

API 응답/요청의 JSON 형태를 클래스로 못박는 도구. **Dify가 전반에서 사용 중** (`account_service.py`, `app_dsl_service.py`, `billing_service.py` 등).

3가지 핵심 가치:
1. **자동 검증** — 타입/필드 누락은 객체 생성 시 즉시 에러 (런타임 위로 미루지 않음)
2. **JSON 직렬화** — `model_dump()` 한 줄로 dict 반환
3. **타입 힌트 = 문서** — IDE 자동완성, 프론트 TypeScript 타입과 1:1 대응

## 상세

### 1. 기본 사용

```python
from pydantic import BaseModel

class KpiDetail(BaseModel):
    count: int
    diff_percent: float | None = None     # Optional (Python 3.10+)
    diff_label: str | None = None

# 정상
detail = KpiDetail(count=47, diff_percent=12.5)

# 자동 에러
KpiDetail(count="47")           # ValidationError: count must be int
KpiDetail(diff_percent=12.5)    # ValidationError: count missing
```

### 2. 중첩 모델

```python
class TotalObjects(KpiDetail):           # 상속 가능
    app_count: int
    kb_count: int
    tool_count: int

class KpiResponse(BaseModel):
    total_objects: TotalObjects
    active_users: KpiDetail
    api_calls: KpiDetail
```

### 3. JSON 직렬화

```python
response = KpiResponse(
    total_objects=TotalObjects(count=47, app_count=32, kb_count=10, tool_count=5),
    active_users=KpiDetail(count=128, diff_percent=12.5),
    api_calls=KpiDetail(count=15420, diff_percent=-8.3),
)

# dict 반환
response.model_dump()
# {"total_objects": {"count": 47, ...}, ...}

# JSON 친화 형태 (datetime → ISO, UUID → str 자동)
response.model_dump(mode='json')

# JSON 문자열 직접
response.model_dump_json()
```

### 4. Flask 응답 패턴

```python
from flask_restful import Resource

class DashboardKpiApi(Resource):
    def get(self):
        response: KpiResponse = DashboardKpiService.get_kpi(...)
        return response.model_dump(mode='json')
```

### 5. 필드 제약

```python
from pydantic import BaseModel, Field

class ErrorRate24h(BaseModel):
    rate: float = Field(ge=0, le=100)        # 0~100 범위 강제
    fail_count: int = Field(ge=0)             # 음수 금지
    total_count: int = Field(ge=0)
    diff_pp: float | None = None
    diff_label: str | None = None
```

### 6. 커스텀 검증

```python
from pydantic import BaseModel, model_validator

class ErrorRate24h(BaseModel):
    rate: float
    fail_count: int
    total_count: int

    @model_validator(mode='after')
    def check_consistency(self):
        if self.total_count == 0 and self.rate != 0:
            raise ValueError("total_count=0이면 rate=0이어야 함")
        return self
```

### 7. 역직렬화 (JSON → Pydantic)

```python
# dict → 모델
data = {"count": 47, "diff_percent": 12.5}
detail = KpiDetail(**data)
# 또는
detail = KpiDetail.model_validate(data)

# JSON 문자열 → 모델
detail = KpiDetail.model_validate_json('{"count": 47, "diff_percent": 12.5}')
```

## dataclass vs Pydantic

| 항목 | `@dataclass` (표준) | Pydantic |
|------|---------|----------|
| 위치 | 표준 라이브러리 | 외부 (`pydantic`) |
| 타입 검증 | ❌ 없음 (타입 힌트만) | ✅ 자동 |
| JSON 직렬화 | ⚠️ `asdict()`만 (limited) | ✅ `model_dump()` |
| 커스텀 검증 | ❌ 직접 구현 | ✅ `@model_validator` |
| 성능 | 빠름 | 약간 느림 (검증 비용) |

→ **API 응답/요청은 Pydantic, 내부 단순 구조체는 dataclass**

## 흔한 함정

### Optional 필드 문법

```python
# Python 3.10+ (권장)
name: str | None = None

# Python 3.9 이하 (Dify는 신문법 사용)
from typing import Optional
name: Optional[str] = None
```

### 디폴트값 = None vs 미지정

```python
class Foo(BaseModel):
    a: int                # 필수
    b: int | None         # 필수, None 허용
    c: int | None = None  # 선택, 미지정 시 None
```

`b`는 명시적으로 `None`을 줘야 함, `c`는 안 줘도 됨.

### datetime 직렬화

```python
from datetime import datetime

class LogEntry(BaseModel):
    created_at: datetime

log = LogEntry(created_at=datetime.now())

log.model_dump()              # → datetime 객체 (직렬화 미완)
log.model_dump(mode='json')   # → "2026-05-04T15:30:00" 문자열 ✅
```

API 응답에는 항상 `mode='json'` 사용.

## SPX-Agent에서의 활용

응답 스키마는 `services/admin/<component>_schemas.py`에 정의:

```python
# services/admin/dashboard_kpi_schemas.py
from pydantic import BaseModel

class KpiDetail(BaseModel):
    count: int
    diff_percent: float | None = None
    diff_label: str | None = None

# ... TypeScript 인터페이스와 1:1 대응
```

각 컴포넌트 design.md "Response 스키마 (Pydantic — 백엔드)" 섹션 참조.

## 관련 노트

- [[4. 지식노트/Dify - Admin API Blueprint 등록 방법.md]]
- [[3. 프로젝트/spx-agent/conventions.md]] § 2 Pydantic 응답 스키마
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md]] (Pydantic 적용 예시)
