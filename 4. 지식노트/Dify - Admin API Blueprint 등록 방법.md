---
tags: [dify, 개발, Flask, 백엔드]
date: 2026-04-29
---
# Dify - Admin API Blueprint 등록 방법

## Blueprint 구조

### 기존 Blueprint 목록

`api/extensions/ext_blueprints.py`에서 등록:

| Blueprint | URL prefix | 용도 |
|-----------|-----------|------|
| `console_app_bp` | `/console/api` | 관리자/사용자 콘솔 API |
| `service_api_bp` | `/api` | 외부 서비스 API |
| `web_bp` | `/` | 웹 애플리케이션 |
| `files_bp` | `/files` | 파일 관리 |
| `inner_api_bp` | - | 내부 API |

### console Blueprint 정의

`api/controllers/console/__init__.py`:
```python
bp = Blueprint("console", __name__, url_prefix="/console/api")
console_ns = Api(bp, ...)
```

**이미 등록되어 있으므로** 새 admin 엔드포인트는 이 Blueprint 안에 모듈만 추가하면 됨.

## 신규 엔드포인트 추가 절차

### Step 1: 컨트롤러 파일 생성

```python
# api/controllers/console/admin/dashboard.py

from controllers.console import console_ns
from flask_restful import Resource

@console_ns.route("/admin/dashboard/kpi")
class DashboardKpiApi(Resource):
    @setup_required
    @login_required
    @account_initialization_required
    def get(self):
        # 구현
        pass
```

### Step 2: import 등록

`api/controllers/console/__init__.py`의 import 목록에 추가하거나, `RESOURCE_MODULES`에 모듈 경로 추가 → 자동 로드

### Step 3: 끝

`console_bp`는 이미 `ext_blueprints.py`에서 등록되어 있으므로, 추가 등록 불필요.

## 라우트/컨트롤러 패턴

### Resource 클래스

```python
@console_ns.route("/admin/dashboard/kpi")
class DashboardKpiApi(Resource):
    @console_ns.doc("get_dashboard_kpi")
    @console_ns.response(200, "Success")
    @setup_required
    @login_required
    @account_initialization_required
    def get(self):
        # HTTP 메서드명 = 메서드 함수명
        pass

    def post(self):
        pass
```

- 경로 파라미터: `<uuid:app_id>`, `<string:name>`
- HTTP 메서드: `get()`, `post()`, `put()`, `delete()`

### 인증/권한 데코레이터 체인

```python
@setup_required                    # 1. 초기화 완료 확인
@login_required                    # 2. 로그인 확인
@account_initialization_required   # 3. 계정 초기화 확인
@edit_permission_required          # 4. 편집 권한 (선택)
@admin_required                    # 5. Admin API Key 검증 (선택)
```

> 데코레이터는 **아래→위** 순서로 실행 (가장 아래 = 가장 먼저)

### method_decorators 패턴

```python
class DashboardKpiApi(Resource):
    method_decorators = [
        account_initialization_required,
        login_required,
        setup_required,
    ]
    # 모든 HTTP 메서드에 자동 적용
```

## 기존 Admin 컨트롤러 참고

| 파일 | 내용 |
|------|------|
| `api/controllers/console/admin.py` | 기존 admin 엔드포인트 (505줄), `admin_required` 데코레이터 사용 |
| `api/controllers/console/apikey.py` | API Key 관리, `method_decorators` 패턴 예시 |
| `api/extensions/ext_blueprints.py` | Blueprint 등록 + CORS 설정 |
| `api/controllers/console/__init__.py` | `bp`, `console_ns` 정의 |

## 관련 노트

- [[4. 지식노트/Dify - 새 API 엔드포인트 등록 방법.md]]
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
