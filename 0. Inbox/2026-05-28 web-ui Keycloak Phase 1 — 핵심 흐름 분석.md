---
tags: [프로젝트, web-ui, keycloak, 분석, phase-1]
date: 2026-05-28
related:
  - "[[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]]"
---

# Phase 1 — 핵심 흐름 분석

> "로그인 요청이 들어오면 어떤 순서로 처리되는가"
> 분석 대상: `keycloak.py` → `passport.py` → `token.py`

---

## 1. ROPC 로그인 (`keycloak.py:202-286` — KeycloakPasswordLoginApi)

### 처리 순서

```
POST /console/api/keycloak/login-password
  ↓
① request body에서 username, password 추출
  ↓
② Keycloak token endpoint 호출
   URL: {KEYCLOAK_INTERNAL_URL}/realms/{REALM}/protocol/openid-connect/token
   body: grant_type=password, client_id, username, password, scope=openid email profile
   timeout: 10초
  ↓
③ 응답 실패(!=200) → AuthenticationFailedError 예외 raise
   (프론트엔드가 인식하는 표준 에러 포맷 {code, message, status} 반환)
  ↓
④ 응답 성공 → access_token, refresh_token 추출
  ↓
⑤ PassportService().verify_keycloak_token(access_token)
   → JWKS RS256 서명 검증 + issuer 검증
  ↓
⑥ decoded에서 sub claim 확인 (없으면 401)
  ↓
⑦ AccountService.provision_default_workspace_for_keycloak(decoded)
   → 계정 JIT 프로비저닝 (upsert by sub)
  ↓
⑧ make_response({"result": "success"})
  ↓
⑨ cookie 3종 설정:
   - set_access_token_to_cookie(access_token)
   - set_refresh_token_to_cookie(refresh_token)
   - set_csrf_token_to_cookie(generate_csrf_token(account.id))
  ↓
⑩ response 반환
```

### 핵심 코드 패턴

```python
# Keycloak 내부 URL 결정
def _keycloak_internal_url() -> str:
    return dify_config.KEYCLOAK_INTERNAL_URL or dify_config.KEYCLOAK_URL

# 토큰 요청
token_response = http_requests.post(
    token_url,
    data={
        "grant_type": "password",
        "client_id": dify_config.KEYCLOAK_CLIENT_ID,
        "username": username,
        "password": password,
        "scope": "openid email profile",
    },
    timeout=10,
)

# 실패 시 표준 에러 형식 반환 (raw JSON이 아님!)
if token_response.status_code != 200:
    raise AuthenticationFailedError()
```

### web-ui 이식 시 주의

- spx-agent는 `flask_restx.Resource` 클래스 기반. web-ui는 일반 Flask Blueprint이므로 함수 기반으로 변환
- `AuthenticationFailedError`는 spx-agent 전용 에러 클래스 → web-ui에서는 단순 JSON 에러 응답으로 대체 가능
- `AccountService.provision_default_workspace_for_keycloak` → web-ui에는 workspace 개념 없음, **생략 또는 단순 세션 저장으로 대체**

---

## 2. 토큰 갱신 (`keycloak.py:289-346` — KeycloakRefreshApi)

### 처리 순서

```
POST /console/api/keycloak/refresh
  ↓
① cookie에서 refresh_token 추출 (extract_refresh_token)
  ↓
② Keycloak token endpoint에 grant_type=refresh_token 호출
  ↓
③ 새 access_token 검증 (verify_keycloak_token)
  ↓
④ sub로 Account 조회 (DB)
  ↓
⑤ cookie 3종 재설정 (access + refresh + csrf)
  ↓
⑥ {"result": "success"} 반환
```

### web-ui 이식 시 주의

- spx-agent는 DB에서 Account를 조회하여 CSRF subject로 사용
- web-ui에서는 DB 조회 대신 **토큰 claims에서 직접 사용자 식별자 추출** 가능 (sub를 그대로 CSRF subject로 사용)

---

## 3. 로그아웃 (`keycloak.py:349-404` — KeycloakLogoutApi)

### 처리 순서

```
POST /console/api/keycloak/logout
  ↓
① cookie에서 refresh_token 추출
  ↓
② Keycloak logout endpoint에 token revoke 요청
   URL: {KEYCLOAK_INTERNAL_URL}/realms/{REALM}/protocol/openid-connect/logout
   body: client_id, refresh_token
   timeout: 5초
   (실패해도 warning만 → 로그아웃 진행)
  ↓
③ cookie 3종 clear:
   - clear_access_token_from_cookie
   - clear_refresh_token_from_cookie
   - clear_csrf_token_from_cookie
  ↓
④ {"result": "success", "redirect_url": kc_logout_url} 반환
```

### web-ui 이식 시 차이 (옵션 B)

설계에서 **옵션 B 채택**: redirect_url 반환 불필요. cookie clear + `{"result": "success"}`만 반환.

```python
# spx-agent 원본: redirect_url 포함
response = make_response({"result": "success", "redirect_url": kc_logout_url})

# web-ui 이식: redirect_url 제거
response = make_response({"result": "success"})
```

---

## 4. JWKS 토큰 검증 (`passport.py` — 전체 95줄)

### 아키텍처

```
                    ┌─────────────────────────────┐
                    │  모듈 레벨 싱글턴            │
                    │  _jwk_client (PyJWKClient)   │
                    │  - cache_keys=True           │
                    │  - lifespan=3600 (1시간)     │
                    │  - thread-safe (Lock)        │
                    └──────────┬──────────────────┘
                               │
              verify_keycloak_token(token)
                               │
                    ┌──────────▼──────────────────┐
                    │ ① JWKS에서 signing key 추출  │
                    │ ② RS256 디코드 (iss/aud 스킵)│
                    │ ③ 수동 issuer 검증           │
                    │   - public_issuer 허용       │
                    │   - internal_issuer 허용     │
                    │ ④ decoded claims 반환        │
                    └─────────────────────────────┘
```

### 핵심 설계 결정

1. **싱글턴 PyJWKClient**: 매 요청마다 생성하면 gevent 환경에서 HTTPS 핸드셰이크 충돌 발생 → 모듈 레벨 + Lock으로 해결
2. **issuer 이중 허용**: `KEYCLOAK_URL`(public)과 `KEYCLOAK_INTERNAL_URL`(internal) 둘 다 허용. Keycloak dev 모드에서 issuer가 요청 hostname 기반으로 변동하기 때문
3. **audience 검증 스킵**: `verify_aud=False` — Keycloak audience 설정이 프로젝트마다 다를 수 있어서

### web-ui 이식 시 주의

- **그대로 이식 가능**. 가장 변경 적은 모듈
- `dify_config` 참조만 web-ui 환경변수 로딩으로 교체
- web-ui에서는 **KEYCLOAK_ISSUER 환경변수**를 사용할 예정 (설계 노트) → issuer 검증 로직을 이에 맞게 조정

---

## 5. Cookie 관리 (`token.py` — 237줄 중 핵심 부분)

### Cookie 3종 스펙

| Cookie | httpOnly | Secure | SameSite | Max-Age | 비고 |
|--------|----------|--------|----------|---------|------|
| `access_token` | ✅ | 조건부 | Lax | ACCESS_TOKEN_EXPIRE_MINUTES × 60 | JS에서 읽을 수 없음 |
| `refresh_token` | ✅ | 조건부 | Lax | REFRESH_TOKEN_EXPIRE_DAYS × 86400 | JS에서 읽을 수 없음 |
| `csrf_token` | ❌ | 조건부 | Lax | ACCESS_TOKEN_EXPIRE_MINUTES × 60 | JS에서 읽어서 헤더로 전송 |

### Secure 조건 판정

```python
def is_secure() -> bool:
    return CONSOLE_WEB_URL.startswith("https") and CONSOLE_API_URL.startswith("https")
```

→ web-ui 배포 도메인 `https://n8n159.spelix.co.kr/`이므로 **Secure=True**

### Cookie 이름 접두사

```python
def _real_cookie_name(cookie_name: str) -> str:
    if is_secure() and _cookie_domain() is None:
        return "__Host-" + cookie_name  # 보안 강화 접두사
    else:
        return cookie_name
```

### Cookie 상수값

```python
COOKIE_NAME_ACCESS_TOKEN = "access_token"
COOKIE_NAME_REFRESH_TOKEN = "refresh_token"
COOKIE_NAME_CSRF_TOKEN = "csrf_token"
HEADER_NAME_CSRF_TOKEN = "X-CSRF-Token"
```

### CSRF 토큰 생성/검증

```python
# 생성: HS256 JWT (subject=user_id, exp=access_token과 동일 수명)
def generate_csrf_token(user_id: str) -> str:
    payload = {"exp": ..., "sub": user_id}
    return PassportService().issue(payload)  # HS256

# 검증: 헤더의 X-CSRF-Token == cookie의 csrf_token + JWT 디코드 후 sub 일치 확인
def check_csrf_token(request, user_id):
    csrf_token = extract_csrf_token(request)         # 헤더에서
    csrf_token_from_cookie = extract_csrf_token_from_cookie(request)  # 쿠키에서
    # 둘이 일치 + JWT 디코드 후 sub == user_id
```

### web-ui 이식 시 주의

- **CSRF는 선택적**. web-ui는 SameSite=Lax + httpOnly cookie 조합으로 기본 CSRF 방어가 되므로, 초기에는 CSRF 토큰 생략 가능
- 단, spx-agent 패턴 그대로 이식하면 보안성이 더 높음
- `__Host-` 접두사는 COOKIE_DOMAIN이 없을 때만 적용 → web-ui 환경에 맞게 판단 필요

---

## Phase 1 요약 — 이식 핵심 판단

| 모듈 | 이식 난이도 | 변경 필요 수준 |
|------|-----------|--------------|
| ROPC 로그인 | 낮음 | AccountService 제거, Blueprint 함수 변환 |
| 토큰 갱신 | 낮음 | DB 조회 제거, sub 직접 사용 |
| 로그아웃 | 매우 낮음 | redirect_url 제거 (옵션 B) |
| JWKS 검증 | 매우 낮음 | 거의 그대로 이식 |
| Cookie 관리 | 중간 | CSRF 포함 여부 결정, 상수/config 정리 |
