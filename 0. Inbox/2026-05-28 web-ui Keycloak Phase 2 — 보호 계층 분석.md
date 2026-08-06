---
tags: [프로젝트, web-ui, keycloak, 분석, phase-2]
date: 2026-05-28
related:
  - "[[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]]"
  - "[[0. Inbox/2026-05-28 web-ui Keycloak Phase 1 — 핵심 흐름 분석]]"
---

# Phase 2 — 보호 계층 분석

> "인증된 요청이 기존 API에 어떻게 적용되는가"
> 분석 대상: `login.py` (데코레이터) → `ext_login.py` (요청 로더)

---

## 1. 요청 로더 (`ext_login.py:44-146` — load_user_from_request)

### 역할

Flask-Login의 `@login_manager.request_loader`로 등록. **매 요청마다** 호출되어 `current_user`를 세팅한다.

### Keycloak 활성 시 처리 흐름

```
매 HTTP 요청
  ↓
① extract_access_token(request)
   → cookie 우선, 없으면 Authorization: Bearer 헤더
  ↓
② KEYCLOAK_ENABLED 확인
  ↓ (True)
③ PassportService().verify_keycloak_token(auth_token)
   → JWKS RS256 검증
  ↓
④ decoded에서 sub claim 추출
  ↓
⑤ DB에서 Account 조회: Account.sub == sub
   → 없으면 401 "Account not provisioned"
  ↓
⑥ AccountService.load_logged_in_account(account_id)
   → tenant context 로드
  ↓
⑦ current_user = account (Flask-Login에 반환)
```

### 핵심 주의사항 (CPU 100% 행 방지)

```python
# ⚠️ 절대 이 함수 안에서 return 전에 logger를 호출하면 안 됨!
# IdentityContextFilter가 current_user를 읽음 → LocalProxy가 이 함수를 재호출
# → 무한 재귀 → gevent worker CPU 100% → healthcheck 행
```

이것은 spx-agent에서 실제 발생한 버그. web-ui에서도 동일 패턴 사용 시 주의 필요.

### web-ui 이식 시 단순화

spx-agent는 `console`, `web`, `mcp` 등 여러 blueprint를 분기 처리하지만, web-ui는 **단일 Flask 앱**이므로:

```python
# web-ui 단순화 버전 (의사 코드)
def load_user_from_request(request):
    token = extract_access_token(request)
    if not token:
        return None  # 비인증 요청 → login_required에서 401 처리
    
    decoded = verify_keycloak_token(token)
    sub = decoded.get("sub")
    if not sub:
        raise Unauthorized()
    
    # web-ui는 DB 없이 토큰 claims로 사용자 정보 구성 가능
    return UserInfo(sub=sub, name=decoded.get("name"), email=decoded.get("email"))
```

---

## 2. login_required 데코레이터 (`login.py:51-102`)

### 작동 방식

```python
@wraps(func)
def decorated_view(*args, **kwargs):
    # ① OPTIONS 요청은 통과 (CORS preflight)
    if request.method in EXEMPT_METHODS:
        return func(...)
    
    # ② current_user 확인
    user = _resolve_current_user()
    if user is None or not user.is_authenticated:
        return login_manager.unauthorized()  # 401 JSON 응답
    
    # ③ Flask g에 user 캐싱
    g._login_user = user
    
    # ④ CSRF 토큰 검증
    check_csrf_token(request, user.id)
    
    # ⑤ 원래 뷰 함수 실행
    return func(...)
```

### current_user 프록시

```python
# LocalProxy로 lazy 로딩
current_user = LocalProxy(lambda: _get_user())

def _get_user():
    if has_request_context():
        if "_login_user" not in g:
            login_manager.load_user_from_request_context()
        return g._login_user
    return None
```

### unauthorized 응답 형식

```python
# DifyLoginManager.unauthorized_handler
Response(
    json.dumps({"code": "unauthorized", "message": "Unauthorized."}),
    status=401,
    content_type="application/json",
)
```

### web-ui 이식 시 단순화

spx-agent는 Flask-Login 전체 인프라(LoginManager, signals, 등)를 사용하지만, web-ui에서는 **경량 데코레이터 하나**로 충분:

```python
# web-ui용 단순 인증 데코레이터 (의사 코드)
from functools import wraps
from flask import request, jsonify, g

def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = extract_access_token(request)
        if not token:
            return jsonify({"code": "unauthorized", "message": "Unauthorized"}), 401
        
        try:
            decoded = verify_keycloak_token(token)
        except Exception:
            return jsonify({"code": "unauthorized", "message": "Unauthorized"}), 401
        
        # role 검증: demo-user 포함 여부
        roles = decoded.get("resource_access", {}) \
                       .get("demo-dify-chat", {}) \
                       .get("roles", [])
        if "demo-user" not in roles:
            return jsonify({"code": "forbidden", "message": "Access denied"}), 403
        
        g.current_user = {
            "sub": decoded["sub"],
            "name": decoded.get("name"),
            "email": decoded.get("email"),
        }
        return f(*args, **kwargs)
    return decorated
```

---

## 3. CSRF 검증 흐름 (`token.py:189-228`)

### Double Submit Cookie 패턴

```
① 로그인 시:
   - csrf_token = JWT(sub=user_id, exp=...) 를 HS256으로 sign
   - Set-Cookie: csrf_token (httpOnly=False → JS 접근 가능)

② API 요청 시:
   - 프론트엔드: cookie에서 csrf_token 읽어서 X-CSRF-Token 헤더로 전송
   - 백엔드 검증:
     a. 헤더의 X-CSRF-Token == cookie의 csrf_token (일치?)
     b. JWT 디코드 후 sub == current_user.id (일치?)
     c. exp 만료 안 됨?
```

### web-ui에서의 CSRF 필요성

| 조건 | web-ui 상태 | CSRF 필요? |
|------|------------|-----------|
| Cookie 기반 인증 | ✅ (httpOnly cookie) | 원칙상 필요 |
| SameSite=Lax | ✅ | 대부분 방어됨 |
| 크로스 도메인 요청 | ❌ (단일 도메인) | 위험 낮음 |

**결론**: SameSite=Lax만으로도 데모 환경에서 충분하지만, spx-agent 패턴 그대로 이식하면 보안 수준이 올라감. **초기 구현에서는 CSRF 생략, 향후 필요 시 추가** 권장.

---

## 4. 비인증 라우트 목록 (web-ui 기준)

현재 web-ui Flask 백엔드의 5개 Blueprint 중 인증이 필요 없는 엔드포인트:

| 엔드포인트 | 인증 필요? | 이유 |
|-----------|-----------|------|
| `GET /health` | ❌ | 헬스체크 |
| `POST /api/auth/login-password` | ❌ | 로그인 자체 |
| `POST /api/auth/refresh` | ❌ | 토큰 갱신 (refresh_token cookie로 인증) |
| `POST /api/auth/logout` | ❌ | 로그아웃 |
| 나머지 모든 API | ✅ | `@login_required` 적용 |

---

## Phase 2 요약 — 이식 핵심 판단

| 항목 | spx-agent | web-ui 이식 |
|------|----------|------------|
| Flask-Login | 전체 인프라 사용 | **불필요** — 경량 데코레이터로 대체 |
| request_loader | blueprint별 복잡 분기 | **단일 경로** — token 추출 → JWKS 검증 → claims 저장 |
| current_user | LocalProxy + Flask g | `g.current_user` dict로 단순화 |
| CSRF | Double Submit Cookie | **초기 생략**, SameSite=Lax로 방어 |
| role 검증 | 없음 (spx-agent는 별도 RBAC) | **데코레이터에 내장** — `resource_access.demo-dify-chat.roles` 확인 |
| unauthorized 응답 | `{"code": "unauthorized", ...}` | 동일 형식 유지 |
