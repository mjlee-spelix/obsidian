---
tags: [프로젝트, web-ui, keycloak, 이식계획, 차이분석]
date: 2026-05-28
related:
  - "[[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]]"
  - "[[0. Inbox/2026-05-28 web-ui Keycloak Phase 1 — 핵심 흐름 분석]]"
---

# web-ui Keycloak 이식 계획 vs 설계 노트 차이 정리

> spx-agent 소스 분석 후 작성한 **이식 계획 (Step 1–10)**과 **설계 노트 PR #2 작업 단계**를 대조한 결과.
> 누락/축소/선행 판단된 항목 6건을 정리하고, 각각에 대해 결정이 필요한 사항을 명시한다.

---

## 이식 계획 전체 (Step 1–10)

### 전체 구조 — spx-agent → web-ui 매핑

```
spx-agent (참고)                    web-ui (이식 결과)
──────────────────                  ──────────────────
keycloak.py (405줄, 6 API)    →    backend/routes/auth.py (~150줄, 4 API)
passport.py (95줄)            →    backend/libs/keycloak_auth.py (~60줄)
token.py (237줄)              →    backend/libs/token.py (~80줄)
login.py (119줄)              →    backend/libs/auth_decorator.py (~30줄)
ext_login.py (179줄)          →    생략 (Flask-Login 미사용)
account_service.py            →    생략 (DB 프로비저닝 불필요)
wraps.py                      →    생략 (이중 모드 불필요)
keycloak_admin.py             →    생략 (Admin API 미사용)
```

총 **~320줄** 수준으로 경량화 (spx-agent 1,000줄+ 대비 약 1/3).

### Step 1: 백엔드 환경변수 + 설정

**파일**: `backend/config.py` (수정)

```python
# 추가할 환경변수
KEYCLOAK_URL = os.getenv('KEYCLOAK_URL', 'http://192.168.10.194:8080')
KEYCLOAK_INTERNAL_URL = os.getenv('KEYCLOAK_INTERNAL_URL', '') or KEYCLOAK_URL
KEYCLOAK_REALM = os.getenv('KEYCLOAK_REALM', 'Spelix')
KEYCLOAK_ISSUER = os.getenv('KEYCLOAK_ISSUER', f'{KEYCLOAK_URL}/realms/{KEYCLOAK_REALM}')
KEYCLOAK_CLIENT_ID = os.getenv('KEYCLOAK_CLIENT_ID', 'demo-dify-chat')
ACCESS_TOKEN_EXPIRE_MINUTES = int(os.getenv('ACCESS_TOKEN_EXPIRE_MINUTES', '60'))
REFRESH_TOKEN_EXPIRE_DAYS = int(os.getenv('REFRESH_TOKEN_EXPIRE_DAYS', '30'))
```

### Step 2: JWKS 검증 모듈

**파일**: `backend/libs/keycloak_auth.py` (신규, ~60줄)

- spx-agent `passport.py`에서 `verify_keycloak_token()` 이식
- PyJWKClient 싱글턴 + 캐싱 (1시간)
- issuer 검증: `KEYCLOAK_ISSUER` 환경변수와 비교
- role 검증 함수 추가: `resource_access['demo-dify-chat'].roles`에 `demo-user` 포함 여부

```python
def verify_keycloak_token(token: str) -> dict:
    """JWKS RS256 검증 + issuer 확인. 실패 시 예외."""

def check_demo_role(decoded: dict) -> bool:
    """decoded claims에서 demo-user role 포함 여부 확인."""
    roles = decoded.get("resource_access", {}) \
                   .get(KEYCLOAK_CLIENT_ID, {}) \
                   .get("roles", [])
    return "demo-user" in roles
```

### Step 3: Cookie 관리 모듈

**파일**: `backend/libs/token.py` (신규, ~80줄)

- spx-agent `token.py`에서 핵심만 이식
- Cookie 3종: `access_token`, `refresh_token`, `csrf_token`
- extract / set / clear 함수 세트
- CSRF는 초기에 생략 — SameSite=Lax로 방어. 나중에 추가 가능

```python
def extract_access_token(request) -> str | None
def set_access_token_to_cookie(response, token)
def set_refresh_token_to_cookie(response, token)
def clear_auth_cookies(response)
```

### Step 4: 인증 데코레이터

**파일**: `backend/libs/auth_decorator.py` (신규, ~30줄)

- Flask-Login 대신 경량 데코레이터 1개
- token 추출 → JWKS 검증 → role 확인 → `g.current_user` 세팅

```python
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
        if not check_demo_role(decoded):
            return jsonify({"code": "forbidden", "message": "Access denied"}), 403
        
        g.current_user = {
            "sub": decoded["sub"],
            "name": decoded.get("name"),
            "email": decoded.get("email"),
        }
        return f(*args, **kwargs)
    return decorated
```

### Step 5: 인증 라우트 Blueprint

**파일**: `backend/routes/auth.py` (신규, ~150줄)

| 엔드포인트 | 메서드 | 출처 (spx-agent) | 변경사항 |
|-----------|--------|-----------------|---------|
| `/api/auth/login-password` | POST | `keycloak.py:202-286` | AccountService 제거, role 검증 추가, 응답 단순화 |
| `/api/auth/refresh` | POST | `keycloak.py:289-346` | DB 조회 제거, sub 직접 사용 |
| `/api/auth/logout` | POST | `keycloak.py:349-404` | redirect_url 제거 (옵션 B) |
| `/api/auth/me` | GET | **신규** | cookie에서 token 추출 → 검증 → 사용자 정보 반환 |

`/api/auth/me` 추가 이유: 프론트엔드에서 세션 유효성 + 사용자 정보 확인용.

### Step 6: 기존 라우트에 인증 적용

**파일**: 기존 5개 Blueprint (`chat.py`, `document.py`, `files.py`, `rooms.py`, `schedules.py`)

각 라우트 함수에 `@login_required` 데코레이터 추가:

```python
from libs.auth_decorator import login_required

@rooms_bp.route('/api/rooms', methods=['GET'])
@login_required
def get_rooms():
    ...
```

### Step 7: Flask app 설정 변경

**파일**: `backend/app.py` (수정)

```python
# 변경 사항:
# 1. auth_bp 등록 추가
from routes.auth import auth_bp
app.register_blueprint(auth_bp)

# 2. CORS에 배포 도메인 + credentials 추가
CORS(app, 
     origins=['http://localhost:3000', 'https://n8n159.spelix.co.kr'],
     supports_credentials=True)
```

### Step 8: 프론트엔드 로그인 페이지 수정

**파일**: `frontend/app/login/page.tsx` (수정)

| 항목 | 현재 | 변경 |
|------|------|------|
| 필드명 | `email` | `username` (label은 "아이디"로) |
| submit | sessionStorage 저장 | `POST /flask-api/api/auth/login-password` |
| 성공 | `router.push('/')` | 동일 |
| 에러 | 클라이언트 검증만 | 서버 응답 기반 에러 메시지 |
| 토큰 저장 | 없음 | 없음 (httpOnly cookie 자동) |

```typescript
// fetch 호출 예시
const res = await fetch('/flask-api/api/auth/login-password', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',  // ← cookie 송수신 필수!
  body: JSON.stringify({ username, password }),
});
```

### Step 9: 프론트엔드 라우트 보호 수정

**파일**: `frontend/app/(main)/layout.tsx` (수정)

```
현재: sessionStorage.getItem('user') → 없으면 /login
변경: fetch('/flask-api/api/auth/me') → 실패하면 /login
```

### Step 10: 헤더에 사용자명 + 로그아웃 버튼 추가

**파일**: Header 컴포넌트 (수정)

- `/api/auth/me` 응답의 `name` 표시
- 로그아웃 버튼: `POST /flask-api/api/auth/logout` → `/login` 이동

```typescript
const handleLogout = async () => {
  await fetch('/flask-api/api/auth/logout', {
    method: 'POST',
    credentials: 'include',
  });
  router.replace('/login');
};
```

### 의존성 추가

```
pip install PyJWT cryptography requests
```

(`PyJWT` — JWT 디코딩, `cryptography` — RS256 지원)

### 파일 생성/수정 요약

| 구분 | 파일 | 작업 |
|------|------|------|
| **신규** | `backend/libs/keycloak_auth.py` | JWKS 검증 + role 확인 |
| **신규** | `backend/libs/token.py` | Cookie 관리 (set/extract/clear) |
| **신규** | `backend/libs/auth_decorator.py` | `@login_required` 데코레이터 |
| **신규** | `backend/routes/auth.py` | 인증 API 4종 |
| **수정** | `backend/config.py` | Keycloak 환경변수 추가 |
| **수정** | `backend/app.py` | auth Blueprint 등록, CORS 설정 |
| **수정** | `frontend/app/login/page.tsx` | 실제 로그인 API 연동 |
| **수정** | `frontend/app/(main)/layout.tsx` | 세션 확인 방식 변경 |
| **수정** | Header 컴포넌트 | 사용자명 + 로그아웃 버튼 |
| **수정** | 기존 라우트 5개 | `@login_required` 추가 |
| **신규** | `backend/.env` | Keycloak 환경변수 값 |

### 작업 순서

```
Step 1-4: 백엔드 인프라 (config → JWKS → cookie → decorator)
  ↓  독립적으로 단위 테스트 가능
Step 5:   인증 라우트 (login-password, refresh, logout, me)
  ↓  curl로 동작 검증 가능
Step 6-7: 기존 라우트 보호 + app 설정
  ↓  
Step 8-10: 프론트엔드 연동
  ↓
검증 시나리오 실행
```

---

## 설계 노트와의 차이 — 총 6건

### 1. PKCE 엔드포인트 2종 누락

| | 설계 노트 | 이식 계획 |
|---|----------|----------|
| `GET /api/auth/login` | O — "이식하되 데모 UX는 ROPC 우선" | **X — 미포함** |
| `GET /api/auth/callback` | O — 동상 | **X — 미포함** |

**설계 의도**: 엔드포인트 구조만 이식해 두고, 데모에서는 사용하지 않지만 향후 SSO 필요 시 활성화.

**이식 계획 판단**: ROPC만으로 데모 충분 → PKCE 생략하여 코드량 축소 (~60줄 절감).

**결정 필요**: 지금 같이 이식할지, 별도 후속 PR로 분리할지.

---

### 2. force-login 엔드포인트 누락

| | 설계 노트 | 이식 계획 |
|---|----------|----------|
| `GET /api/auth/force-login` | O — "선택" | **X — 미포함** |

**설계 의도**: Keycloak SSO 세션 강제 해제 후 재로그인. PKCE 흐름에서 기존 SSO 세션이 남아있을 때 사용.

**이식 계획 판단**: ROPC 전용 환경에서는 SSO 세션을 사용하지 않으므로 무의미.

**결정 필요**: 설계 노트에서도 "선택"이므로 생략해도 무방할 가능성 높음. 확인 필요.

---

### 3. LLM 호출 토큰 첨부 누락

| | 설계 노트 | 이식 계획 |
|---|----------|----------|
| access_token → LLM 게이트웨이 전달 | O (2항목) | **X — 미포함** |
| LLM 측 검증 필요 여부 | "별도 결정" | **X — 미포함** |

**설계 노트 원문**:
> - 백엔드가 LLM 호출 게이트웨이 역할이면 access_token을 헤더로 전달
> - LLM 측 검증 필요 여부는 별도 결정 (현재는 백엔드만 검증으로 충분 추정)

**이식 계획 판단**: 이식 계획 작성 시 스코프에서 빠짐 (분석 범위가 인증 흐름에 집중).

**결정 필요**: PR #2 범위에 포함할지, 별도 작업으로 분리할지. 설계 노트에서도 "별도 결정"으로 유보한 부분이라 PR #2 이후로 미뤄도 무방할 수 있음.

---

### 4. CSRF cookie — 설계는 포함, 이식 계획은 초기 생략

| | 설계 노트 | 이식 계획 |
|---|----------|----------|
| cookie 3종 (access / refresh / **csrf**) | O — 일관 명시 | **csrf 초기 생략** |

**설계 노트**: "httpOnly cookie 3종 (access / refresh / csrf)"를 통합 전략 표에 명시. PR #2 백엔드 항목에서도 "cookie 3종 설정"으로 기술.

**이식 계획 판단**: SameSite=Lax + httpOnly cookie 조합으로 기본 CSRF 방어 충분 → 초기에는 CSRF 토큰 생략하여 복잡도 축소.

**트레이드오프**:
- 생략 시: cookie 2종만 관리, 프론트엔드에서 `X-CSRF-Token` 헤더 전송 불필요 → 구현 단순
- 포함 시: spx-agent 패턴 그대로 이식, 보안 수준 동일 → 설계 노트와 정합
- SameSite=Lax는 cross-site POST를 차단하므로 단일 도메인 데모에서 CSRF 공격 벡터 사실상 없음

**결정 필요**: 설계대로 3종 유지할지, 이식 계획대로 2종으로 갈지.

---

### 5. 계정 provisioning — 설계는 미결, 이식 계획은 생략 확정

| | 설계 노트 | 이식 계획 |
|---|----------|----------|
| `AccountService.provision_default_workspace_for_keycloak` | "생략 가능 — **검토 필요**" | "**생략**"으로 확정 |

**설계 노트 원문**:
> 계정 provisioning — spx-agent의 `AccountService.provision_default_workspace_for_keycloak`를 web-ui 환경에 맞춰 단순화 (web-ui는 별도 RBAC 없으면 생략 가능 — 검토 필요)

**이식 계획 판단 근거**:
- web-ui에 `accounts` 테이블 없음 (PostgreSQL 스키마가 다름)
- workspace/tenant 개념 없음 (단일 앱)
- JWT claims에서 사용자 정보 직접 추출 가능 (DB 저장 불필요)

**결정 필요**: 이식 계획의 "생략" 판단을 설계 노트에 반영(확정)할지.

---

### 6. 세션 확인 훅 — 설계는 별도 파일, 이식 계획은 inline

| | 설계 노트 | 이식 계획 |
|---|----------|----------|
| 세션 확인 | `lib/auth.ts` 또는 신규 **훅 파일 분리** | `layout.tsx`에서 **직접 fetch** |

**설계 노트 원문**:
> 세션 확인 훅 — `lib/auth.ts` 또는 신규
> `/api/auth/me` 호출로 현재 사용자 정보 가져오기

**이식 계획 판단**: layout.tsx에서 직접 fetch하면 별도 파일 없이 간결.

**트레이드오프**:
- 별도 훅(`useAuth` 등): 여러 컴포넌트에서 재사용 가능 (Header에서 사용자명, layout에서 보호, 등)
- inline: 파일 수 최소화, 단 로직 중복 가능성

**결정 필요**: layout + Header 모두에서 사용자 정보가 필요하므로, 별도 훅이 실용적일 수 있음.

---

## 대조 요약표

| #   | 항목                        | 설계 노트    | 이식 계획        | 차이 유형 | 영향도                  |
| --- | ------------------------- | -------- | ------------ | ----- | -------------------- |
| 1   | PKCE 2종 (login, callback) | 이식하되     | 미포함          | 누락    | 낮음 (데모 미사용)          |
| 2   | force-login               | 선택       | 미포함          | 누락    | 낮음 (ROPC에서 무의미)      |
| 3   | LLM 토큰 첨부                 | 포함 (2항목) | 미포함          | 누락    | 미정 (별도 결정 대상)        |
| 4   | CSRF cookie               | 3종 명시    | 2종 (csrf 생략) | 축소    | 낮음 (SameSite=Lax 방어) |
| 5   | 계정 provisioning           | 검토 필요    | 생략 확정        | 선행 판단 | 낮음 (DB 없음)           |
| 6   | 세션 확인 훅                   | 별도 파일    | inline       | 구조 차이 | 낮음 (기능 동일)           |

→ 코드 동작에 영향을 주는 차이는 **#1 PKCE**와 **#4 CSRF**. 나머지는 구조/스코프 차이.
→ 6건 모두 결정 후 설계 노트 또는 이식 계획 한쪽을 업데이트하여 정합시킬 필요 있음.

---

## 구현 전 추가 고려 사항 3건

### A. 토큰 만료 시 불필요한 재로그인 문제 (영향도: 높음)

현재 계획의 흐름에 UX 결함이 있음:

```
사용자가 61분 후 페이지 이동
→ layout의 useAuth()가 /api/auth/me 호출
→ access_token 만료 → 401
→ /login으로 redirect
→ 그런데 refresh_token은 30일 유효!
→ 불필요한 재로그인 발생
```

**대응 필요**: 프론트엔드에서 401 응답 시 자동 갱신 로직 추가.

**선택지**:

| 방식 | 구현 위치 | 장점 | 단점 |
|------|----------|------|------|
| A. fetch 래퍼에서 401 → refresh → 재시도 | 프론트엔드 (fetch 래퍼 or useAuth 훅) | 표준적, 모든 API에 일괄 적용 | fetch 래퍼 신규 작성 필요 |
| B. `/api/auth/me`에서 백엔드가 자동 갱신 | 백엔드 `/me` 엔드포인트 | 프론트엔드 단순 | `/me` 외 다른 API는 여전히 401 |
| C. 프론트엔드 타이머로 사전 갱신 | 프론트엔드 (setInterval) | 만료 전에 갱신 | 타이머 관리 복잡, 탭 비활성 시 동작 불확실 |

**권장**: **방식 A** — fetch 래퍼에 401 intercept + refresh 로직 내장. `useAuth` 훅에서도 동일 래퍼 사용.

```typescript
// 의사 코드
async function authFetch(url, options) {
  let res = await fetch(url, { ...options, credentials: 'include' });
  
  if (res.status === 401) {
    const refreshRes = await fetch('/flask-api/api/auth/refresh', {
      method: 'POST', credentials: 'include'
    });
    if (refreshRes.ok) {
      // refresh 성공 → 원래 요청 재시도
      res = await fetch(url, { ...options, credentials: 'include' });
    } else {
      // refresh도 실패 → 로그인으로
      window.location.href = '/login';
    }
  }
  return res;
}
```

**결정 필요**: 방식 선택 + 이식 계획 Step 8-10에 반영 여부.

---

### B. 운영 환경 Cookie Secure 플래그 판정 (영향도: 중간)

spx-agent는 `CONSOLE_WEB_URL.startsWith("https")`로 Secure 플래그 설정 여부를 판정. web-ui에는 이 변수가 없음.

또한 Flask가 gunicorn 뒤에서 동작하므로 `request.is_secure`가 `False`를 반환할 수 있음 (리버스 프록시가 `X-Forwarded-Proto` 헤더를 안 넘기면).

| 선택지 | 방식 | 장점 | 단점 |
|--------|------|------|------|
| A. 환경변수 `COOKIE_SECURE=true` | config.py에 추가 | 명시적, 단순 | 환경변수 1개 추가 |
| B. Flask `ProxyFix` 미들웨어 | app.py에 적용 | `request.is_secure` 자동 판정 | 프록시 설정 의존 |

**결정 필요**: Step 3 (token.py) 구현 시 Secure 플래그 판정 방식 확정.

---

### C. `backend/libs/` 디렉토리 미존재 (영향도: 낮음)

현재 프로젝트에 `backend/libs/` 디렉토리가 없음. Step 2-4에서 신규 파일 3개를 이 경로에 생성하므로:

- `backend/libs/` 디렉토리 생성
- `backend/libs/__init__.py` 생성 (Python 패키지 인식용)

`requirements.txt`에는 `requests`가 이미 포함. `PyJWT`와 `cryptography`만 추가하면 됨.

---

### 기타 확인 결과 (문제 없음)

| 항목 | 상태 | 비고 |
|------|------|------|
| Next.js rewrite → cookie 전달 | ✅ | same-origin이라 `Set-Cookie` 정상 통과 |
| gunicorn + PyJWKClient 싱글턴 | ✅ | sync worker(기본) → threading.Lock 정상 동작 |
| 기존 `/flask-api` 호출 패턴 | ✅ | 3개 컴포넌트에서 사용 중, 동일 패턴으로 auth API 호출 가능 |
| sales-support 별도 인증 | ✅ | localStorage 기반 → Keycloak cookie와 충돌 없음 |
| 운영 Dockerfile | ✅ | gunicorn 포함, `requirements.txt` 기반 설치 |
