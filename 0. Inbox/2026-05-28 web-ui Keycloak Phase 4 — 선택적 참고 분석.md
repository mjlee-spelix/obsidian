---
tags: [프로젝트, web-ui, keycloak, 분석, phase-4]
date: 2026-05-28
related:
  - "[[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]]"
  - "[[0. Inbox/2026-05-28 web-ui Keycloak Phase 1 — 핵심 흐름 분석]]"
---

# Phase 4 — 선택적 참고 분석

> "web-ui에서 생략/단순화할 부분 판단"
> 분석 대상: `account_service.py`, `wraps.py`, configs

---

## 1. 계정 프로비저닝 (`account_service.py:132-199`)

### spx-agent 원본 — JIT Provisioning

Keycloak 로그인 성공 시마다 호출. idempotent (upsert by sub).

```python
def provision_default_workspace_for_keycloak(claims: dict) -> Account:
    sub = claims["sub"]
    email = claims.get("email") or sub
    name = claims.get("name") or claims.get("preferred_username") or email

    # 1. Account upsert by sub
    account = db.session.scalar(select(Account).where(Account.sub == sub))
    if account is None:
        account = Account(name=name, email=email)
        account.sub = sub
        # ... 초기 설정
        db.session.add(account)
        db.session.flush()
    else:
        # IdP 소유 필드 동기화 (매 로그인마다)
        account.email = email
        account.name = name
        account.interface_language = locale

    # 2. Workspace 멤버십 보장
    existing_join = db.session.scalar(...)
    if existing_join is None:
        # 첫 번째 사용자 → owner, 이후 → normal member
        ...
```

### web-ui 이식 판단: **생략**

| 이유 | 설명 |
|------|------|
| DB 스키마 차이 | web-ui에는 `accounts` 테이블 없음 (PostgreSQL 스키마가 다름) |
| Workspace 개념 없음 | web-ui는 단일 앱, 멀티 테넌시 불필요 |
| 사용자 정보 출처 | JWT claims에서 직접 추출하면 충분 |

### 대체 방안

로그인 성공 시 JWT claims를 그대로 사용:

```python
# web-ui: DB 프로비저닝 없이 claims 직접 활용
decoded = verify_keycloak_token(access_token)
user_info = {
    "sub": decoded["sub"],
    "name": decoded.get("name", ""),
    "email": decoded.get("email", ""),
}
# → cookie에 저장된 access_token에서 매 요청마다 추출
```

---

## 2. 조건부 데코레이터 (`wraps.py`)

### spx-agent 패턴

```python
# ① setup_required — Keycloak 모드에서는 스킵
def setup_required(view):
    if dify_config.KEYCLOAK_ENABLED:
        return view  # IdP가 install wizard 대체
    # ... DifySetup 확인 로직

# ② keycloak_disabled — Keycloak 모드에서 이메일/비밀번호 로그인 차단
def keycloak_disabled(view):
    if dify_config.KEYCLOAK_ENABLED:
        abort(404)
    return view(...)
```

### web-ui 이식 판단: **불필요**

- web-ui는 처음부터 Keycloak만 사용 (이중 인증 모드 없음)
- `KEYCLOAK_ENABLED` 토글 불필요 — 항상 Keycloak
- `setup_required` 개념 없음

---

## 3. 환경변수 설정 (`configs/feature/__init__.py`)

### spx-agent 환경변수 (Keycloak 관련)

```ini
KEYCLOAK_ENABLED=true
KEYCLOAK_URL=http://192.168.10.194:8080          # public (브라우저용)
KEYCLOAK_INTERNAL_URL=http://192.168.10.194:8080  # internal (백엔드용)
KEYCLOAK_REALM=Spelix
KEYCLOAK_CLIENT_ID=dify-app
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=admin
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30
COOKIE_DOMAIN=
```

### web-ui 환경변수 (설계 노트 기준)

```ini
# --- Keycloak ---
KEYCLOAK_URL=http://192.168.10.194:8080
KEYCLOAK_INTERNAL_URL=http://192.168.10.194:8080
KEYCLOAK_REALM=Spelix
KEYCLOAK_ISSUER=http://192.168.10.194:8080/realms/Spelix
KEYCLOAK_CLIENT_ID=demo-dify-chat

# --- Session ---
SESSION_SECRET=<랜덤 생성>
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30

# --- App ---
FLASK_SECRET_KEY=<랜덤 생성>
```

### 차이점

| 항목 | spx-agent | web-ui |
|------|----------|--------|
| `KEYCLOAK_ENABLED` | 있음 (토글) | **없음** (항상 활성) |
| `KEYCLOAK_ISSUER` | 없음 (코드에서 조합) | **있음** (명시적) |
| `KEYCLOAK_CLIENT_ID` | `dify-app` | `demo-dify-chat` |
| `KEYCLOAK_ADMIN*` | 있음 (Admin API 사용) | **없음** (Admin API 미사용) |
| `COOKIE_DOMAIN` | 있음 | 필요 시 추가 |

---

## 4. 기타 참고 사항

### PKCE 흐름 (보조, 현재 미사용)

`keycloak.py:70-109` (KeycloakLoginApi) + `keycloak.py:112-199` (KeycloakCallbackApi)

- 엔드포인트는 이식하되 데모에서는 사용하지 않음
- 향후 외부 SSO 연동 필요 시 활성화 가능
- server-side session에 `code_verifier`, `state` 저장 → Flask `session` 사용

### force-login (선택)

`keycloak.py:43-67` (KeycloakForceLoginApi)

- Keycloak SSO 세션 강제 해제 후 재로그인
- ROPC 전용 환경에서는 의미 없음 (SSO session 안 씀)
- **이식 불필요**

### keycloak_admin.py (미사용)

- Keycloak Admin API 클라이언트 (사용자 조회, 그룹 관리 등)
- web-ui에서는 Admin API 미사용 → **이식 불필요**

---

## Phase 4 요약 — 생략/이식 판단 매트릭스

| 모듈 | spx-agent 용도 | web-ui 판단 | 이유 |
|------|---------------|------------|------|
| `account_service.py` (JIT provisioning) | 매 로그인 시 Account upsert | **생략** | web-ui에 accounts 테이블/workspace 없음 |
| `wraps.py` (setup_required, keycloak_disabled) | 이중 인증 모드 분기 | **생략** | web-ui는 Keycloak 전용 |
| `keycloak_admin.py` | Admin API 호출 | **생략** | 관리 기능 불필요 |
| PKCE 흐름 (login + callback) | SSO redirect 인증 | **보류** | 엔드포인트 구조만 준비, 데모에서 미사용 |
| `force-login` | SSO 세션 강제 해제 | **생략** | ROPC 환경에서 무의미 |
| 환경변수 | pydantic Settings | **단순화** | `os.getenv` + config.py 확장 |
