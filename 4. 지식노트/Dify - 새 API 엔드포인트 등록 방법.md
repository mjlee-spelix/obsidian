---
tags: [dify, 개발, AI-Agent]
date: 2026-04-27
---
# Dify - 새 API 엔드포인트 등록 방법

## 핵심
- 새 API 엔드포인트는 **컨트롤러 파일 생성 → `__init__.py`에 import 추가** 2단계면 라우트 등록 완료
- Flask-RESTful의 `Namespace.route()` 데코레이터를 사용하며, import만 하면 자동 등록되는 구조
- 3개의 API 레이어 존재: **Console API** (`/console/api/`), **Service API** (`/v1/`), **Inner API** (`/inner/api/`)
- 외부 HTTP 호출이 필요하면 `api/services/`에 Service 클래스를 별도로 만들어 분리

## 상세

### 아키텍처 계층 구조

```
Flask App
  └─ Blueprint 등록 (api/extensions/ext_blueprints.py)
       └─ Namespace 정의 (api/controllers/console/__init__.py)
            └─ Controller 파일 (api/controllers/console/app/*.py)
                 └─ Service 호출 (api/services/*.py)
```

| 계층 | 위치 | 역할 |
|------|------|------|
| Blueprint 등록 | `api/extensions/ext_blueprints.py` | Flask 앱에 Blueprint 등록 |
| Namespace 정의 | `api/controllers/console/__init__.py` | URL prefix(`/console/api`) + 컨트롤러 import |
| Controller | `api/controllers/console/app/*.py` | 라우트 + 요청/응답 처리 |
| Service | `api/services/*.py` | 비즈니스 로직, 외부 API 호출 |

### 3개 API 레이어

| API 레이어 | URL Prefix | 용도 | 인증 방식 |
|-----------|-----------|------|----------|
| **Console API** | `/console/api/` | 관리 대시보드용 (우리가 주로 쓸 곳) | `@login_required` (세션/JWT) |
| **Service API** | `/v1/` | 외부 개발자용 | `@validate_app_token` |
| **Inner API** | `/inner/api/` | 내부 서비스 간 통신 | `X-Inner-Api-Key` 헤더 |

> [!tip] 중앙 관리 대시보드는 Console API 레이어에 추가
> 관리자가 로그인 후 사용하는 기능이므로 `/console/api/` prefix 아래에 만든다.

### 새 엔드포인트 추가 절차 (3단계)

#### ① 컨트롤러 파일 생성

`api/controllers/console/app/` 아래에 새 파일 생성.

```python
from flask_restful import Resource
from controllers.console import api as console_app_ns
from controllers.console.wraps import setup_required, login_required, account_initialization_required, get_app_model

class YourNewApi(Resource):
    @setup_required
    @login_required
    @account_initialization_required
    @get_app_model
    def get(self, app_model):
        # 비즈니스 로직 또는 서비스 호출
        result = YourService.do_something(app_model)
        return result

# 라우트 등록 — 이 줄이 핵심!
console_app_ns.add_resource(YourNewApi, "/apps/<uuid:app_id>/your-new-path")
```

#### ② `__init__.py`에 import 추가

`api/controllers/console/__init__.py` 파일 하단에 한 줄 추가:

```python
# 기존 import들...
from .app import statistic, workflow_statistic, ...

# 이 한 줄로 라우트 자동 등록!
from .app import your_new_module
```

> [!important] import만 하면 등록 완료
> Python 모듈이 import되면서 `console_app_ns.add_resource()`가 실행되어 자동으로 라우트가 등록된다. Blueprint에 별도 등록하는 작업은 불필요.

#### ③ (선택) Service 파일 생성

외부 API 호출이나 복잡한 비즈니스 로직은 `api/services/`에 분리.

```python
import httpx
from tenacity import retry, wait_fixed, stop_before_delay

class NewFeatureService:
    base_url = os.environ.get("DIFY_API_BASE_URL")
    secret_key = os.environ.get("DIFY_API_SECRET_KEY")

    @classmethod
    @retry(wait=wait_fixed(2), stop=stop_before_delay(10))
    def _send_request(cls, method, endpoint, json=None, params=None):
        headers = {
            "Content-Type": "application/json",
            "Billing-Api-Secret-Key": cls.secret_key,
        }
        url = f"{cls.base_url}{endpoint}"
        response = httpx.request(method, url, headers=headers, json=json, params=params)
        return response.json()
```

### 주요 데코레이터

| 데코레이터 | 위치 | 용도 |
|-----------|------|------|
| `@setup_required` | `wraps.py` | 시스템 초기 설정 완료 확인 |
| `@login_required` | `wraps.py` | 로그인 상태 확인 (JWT/세션) |
| `@account_initialization_required` | `wraps.py` | 계정 초기화 확인 |
| `@get_app_model` | `wraps.py` | URL의 `app_id` → `app_model` 자동 로드 |
| `@admin_required` | `wraps.py` | 어드민 권한 확인 |

### 참고할 기존 패턴별 파일

| 패턴 | 참고 파일 | 설명 |
|------|----------|------|
| DB 직접 쿼리 | `api/controllers/console/app/statistic.py` | SQLAlchemy로 집계 쿼리 |
| Service 호출 | `api/controllers/console/workspace/workspace.py` | 서비스 레이어 분리 |
| 외부 HTTP API | `api/services/billing_service.py` | httpx + tenacity 재시도 패턴 |

## 관련 노트
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[4. 지식노트/spx-agent - 프로젝트 폴더 구조.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 소스 분석 현황.md]]
