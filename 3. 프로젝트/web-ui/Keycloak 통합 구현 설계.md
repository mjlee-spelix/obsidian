---
tags: [프로젝트, web-ui, 인증, keycloak, 설계]
date: 2026-05-28
status: PR #1·#2 완료 — 배포 + 배포 환경 검증 대기 (5/29)
related:
  - "[[3. 프로젝트/web-ui/로그인 인증 설계]]"
  - "[[0. Inbox/2026-05-26 web-ui Keycloak 통합 사전 분석 위임 (P2)]]"
---

# web-ui Keycloak 통합 구현 설계

> 3/30 [[3. 프로젝트/web-ui/로그인 인증 설계]]에서 미정으로 남았던 "Supabase vs Keycloak" 선택이 **Keycloak으로 확정**되고, 5/26 P2 사전 분석 + 5/28 AAI 주간 보고 회의로 통합 방식이 결정됨. 5/28 추가 검토에서 승랑님 소스 직접 확인 결과 **데모 UX는 ROPC grant 방식**(web-ui 자체 로그인 화면 유지)임이 확인되어 통합 전략이 조정됨. 본 노트는 그 결정 사항과 구현 작업 순서를 정리한다.

## 배경

- **대상 프로젝트**: `C:\Users\Administrator\Projects\n8n\poc\web-ui\dify-chat`
- **현재 상태**: 로그인 화면(`frontend/app/login/page.tsx`)이 form UI + sessionStorage 통과형 stub
- **목표**: 외부 고객 시연용 데모 사이트에 실제 로그인 기능 추가 (Spelix realm 재활용)
- **데모 대상**: **외부 고객** (5/28 회의 확인)
- **배포 도메인**: `https://n8n159.spelix.co.kr/` (확정, 운영 중)
- **참고 구현**: spx-agent의 Keycloak 통합 (승랑님 작업분 6 커밋, `f3e9dbe → ... → 3acfba4`)

## 통합 전략 — SSR + httpOnly cookie + ROPC grant

| 항목 | 내용 |
|---|---|
| 프론트엔드 | Next.js 16.1.6 (App Router) + React 19.2.3 |
| 백엔드 | 별도 Flask (`backend/app.py`, 5 Blueprint) — 인증 라우트는 신규 추가 |
| 주력 인증 흐름 | **Resource Owner Password Grant (ROPC)** — web-ui 자체 폼에서 ID/PW 받아 백엔드가 Keycloak token endpoint 직접 호출 |
| ~~보조 인증 흐름 (PKCE)~~ | **이식하지 않음** — YAGNI. 향후 외부 SSO 필요 시 별도 작업 (~60줄) |
| 토큰 보관 | httpOnly cookie 2종 (access / refresh). CSRF 초기 생략 — SameSite=Lax + httpOnly + 단일 도메인 조합으로 방어. 필요 시 csrf 추가 |
| 토큰 갱신 | refresh_token으로 silent refresh (백엔드 ↔ Keycloak) |
| 로그아웃 | **옵션 B 채택** — 백엔드 token revoke + cookie clear만, Keycloak `/logout` 브라우저 redirect는 생략 |
| 채택 근거 | spx-agent 패턴 핵심만 이식 (~320줄, spx-agent 1,000줄+ 대비 1/3) / React 19 라이브러리 충돌 위험 없음 / Keycloak 외부 도메인 불필요 |

### UX 흐름 — Keycloak 화면 안 보임

```
1. 외부 고객이 https://n8n159.spelix.co.kr/ 접속
2. 비로그인이면 web-ui 자체 로그인 화면 표시 (현재 form UI 그대로)
3. ID/PW 입력 → submit
   → 프론트엔드: POST /api/auth/login-password (credentials: 'include')
   → 백엔드: Keycloak token endpoint에 grant_type=password로 호출
   → Keycloak: 토큰 발급
   → 백엔드: JWKS RS256 검증 + role 확인 + cookie 2종 설정 + 성공 응답
4. 프론트엔드: 홈 화면 진입
```

→ 사용자는 Keycloak 도메인을 한 번도 보지 않음. UX는 기존 stub 화면과 동일.

### ROPC 트레이드오프 (받아들임)

- ROPC는 OAuth2 보안 안티패턴으로 간주됨 (백엔드가 사용자 비밀번호를 직접 핸들링)
- 다만 데모/내부 신뢰 환경에서는 흔히 사용되는 패턴
- 승랑님 spx-agent 구현도 동일 트레이드오프 수용 (`keycloak.py:202-286` `/keycloak/login-password`)
- 외부 공개 OAuth 클라이언트가 아니라 자체 데모 사이트라 위험 통제 가능

## Keycloak 인프라 결정

### realm 및 client

| 항목 | 결정 |
|---|---|
| **realm** | `Spelix` 공유 (spx-agent와 동일) |
| **client** | `demo-dify-chat` 신설 (public, **Direct Access Grants 활성화**) |
| **위치** | 194 Keycloak (`http://192.168.10.194:8080`) |

> client 신설 이유: web-ui는 spx-agent와 별개 앱이므로 자기 신원(client_id)이 따로 필요. redirect_uri / CORS / 접근 제어를 앱 단위로 분리하기 위함.

### 인프라 사전 검증 (5/28 완료)

- ✅ 159 ↔ 194:8080 사내망 통신 가능 (curl HTTP 200)
- ✅ `Spelix` realm 존재 (대소문자 정확)
- ✅ Keycloak realm에 `password` grant 지원 활성화됨 (`grant_types_supported`에 포함)
- ✅ PKCE 지원 (`code_challenge_methods_supported: ["plain", "S256"]`)
- ✅ JWKS endpoint 정상 (`jwks_uri`)
- ✅ 방화벽 / 호스트 firewall / 컨테이너 포트 publish 전부 통과

→ 인프라 작업 0건, PR #1 즉시 착수 가능

### Keycloak 외부 도메인 불필요

ROPC 흐름에서는 **백엔드만 Keycloak에 접근**하면 됨. 사용자 브라우저는 Keycloak 도메인에 접근하지 않음.

| 통신 | 누가 → 누구 | 사내 IP로 충분한가 |
|---|---|---|
| 로그인 (ROPC) | 백엔드(159) → Keycloak(194) | ✅ 사내망 직통 |
| 토큰 갱신 | 백엔드 → Keycloak | ✅ |
| 로그아웃 token revoke | 백엔드 → Keycloak | ✅ |
| ~~로그아웃 SSO cookie 정리~~ | ~~브라우저 → Keycloak~~ | (옵션 B로 생략) |

### 데모 계정 운영 — 신규 발급(A′) + 그룹 미배정

> 5/28 이사님 발언 "dify 계정이어도 ㄱㅊ"는 **신규 발급 부담을 줄이려는 허용 차원**이지 강제는 아님. 추가 검토 결과 다음 이유로 신규 발급으로 결정.

| 항목 | 결정 |
|---|---|
| 신규 Keycloak 사용자 발급 | **함** — `demo-01`, `demo-02`, `demo-03` 3명 신규 생성 |
| 그룹 배정 | **하지 않음** — DEMO 그룹 자체를 만들지 않음 |
| client role 부여 | 사용자에게 **직접 매핑** (Users → demo-XX → Role mapping → `demo-dify-chat-user` 추가) |
| 시작 인원 | 3명 — 동시 시연 2~3건 커버. 부족 시 추가 발급 (5분 작업) |
| 계정 시나리오 분리 | `demo-01` = 시장조사 / `demo-02` = 사규검색 / `demo-03` = 예비 |

**신규 발급으로 결정한 이유**:
- 외부 고객 데모인데 사내 사용자 이름(`김민준` 등) 노출 회피
- 데모 시연자가 시나리오별 계정명(`demo-01`)으로 일관되게 진행 가능
- 데모 계정이 spx-agent에 들어가도 `spx_accounts` 외 다른 테이블 오염 0건

**그룹 미배정으로 결정한 이유**:
- 데모 계정의 JWT `claims.groups`가 비어 있음
- → spx-agent의 `_sync_department_from_keycloak_groups`가 no-op
- → `spx_departments` 영향 0건 (DEMO 부서 자동 생성 트리거 회피)
- 3명이라 role 일괄 관리 이득 미미

**spx-agent 자동 차단 메커니즘** (`controllers/console/workspace/rbac.py:377-414`):

데모 계정이 실수로 spx-agent에 로그인해도 다음 메커니즘으로 메인 화면 진입 차단:
```
1. 백엔드 인증 통과 → accounts + tenant_account_joins 행 생성 (normal role)
2. 프론트엔드 메인 화면 진입 시도
   → GET /workspaces/current/rbac/departments/for-actor
3. 부서 멤버십 없음 + admin 아님 → departments=[] 응답
4. 프론트엔드: no-department onboarding 페이지로 강제 redirect
5. ❌ 메인 화면 진입 불가
```

→ 별도 차단 코드 없이 **자동 차단**됨. DB noise는 `accounts` + `tenant_account_joins` 2행 정도, `spx_departments`는 깨끗.

### client 접근 제어 — Role 직접 부여 방식

> "그 계정으로만 로그인 되게" (5/28 이사님 발언) 요건 충족.

| 방식 | 채택 여부 | 근거 |
|---|---|---|
| Client Authorization (fine-grained) | ✗ | 세밀함이 과함, 설정·디버깅 복잡 |
| 그룹 → role 매핑 | ✗ | DEMO 그룹이 spx-agent JIT 동기화 트리거 (spx_departments 오염) |
| **사용자에게 role 직접 부여** | ✓ | 단순 / 그룹 미사용으로 spx-agent 영향 0 / 3명이라 일괄 관리 이득 미미 |

**구현 그림**:

```
1. demo-dify-chat client에 client role `demo-dify-chat-user` 생성
2. 데모 사용자 demo-01, demo-02, demo-03 신규 생성
3. 각 사용자에 role 직접 매핑 (Users → demo-XX → Role mapping → `demo-dify-chat-user` 추가)
4. web-ui 백엔드: 토큰 검증 시
   resource_access['demo-dify-chat'].roles 에 'demo-dify-chat-user' 포함 여부 확인
   → 없으면 403
```

**spx-agent role claim 미사용 확인** (account_service.py 전수 조사):
- spx-agent는 JWT의 `realm_access` / `resource_access` claims를 **사용 안 함** (검색 결과 0건)
- 사용하는 claim: `sub`, `email`, `name`, `groups`만
- → role 부여해도 spx-agent DB 영향 0

> 차단 범위는 **방향 1만** (비-DEMO 사용자 → demo-dify-chat 차단). 역방향(데모 계정 → spx-agent)은 onboarding 페이지로 자동 차단됨 (위 § "데모 계정 운영" 참조). 별도 코드/정책 추가 불필요.

## 미결정 사항

| # | 항목 | 결정 | 상태 |
|---|---|---|---|
| 1 | client_id 명명 | `demo-dify-chat` | ✅ 해결 (`dify-app`/`dify-audit`는 Dify 베이스 시스템용이라 별개 카테고리로 분리, `demo-` prefix로 데모 명시) |
| 2 | redirect_uri 등록 시점 | dev + 배포 URL 모두 등록 | ✅ 해결 (`https://n8n159.spelix.co.kr/` 확정) |
| 3 | Keycloak 외부 공개 도메인 필요 여부 | 불필요 | ✅ 해결 (ROPC 사용으로 자동 해소) |
| 4 | UX 흐름 (Keycloak 화면 vs web-ui 자체 화면) | web-ui 자체 화면 | ✅ 해결 (승랑님 패턴 = ROPC) |
| 5 | 로그아웃 처리 방식 | 옵션 B (백엔드 revoke + cookie clear, Keycloak redirect 생략) | ✅ 해결 |

→ 모든 결정 완료. PR #1 즉시 착수 가능.

---

## 작업 단계 — PR #1: Keycloak 인프라 셋업

> 194 Keycloak admin UI(`http://192.168.10.194:8080`)에서 진행. 코드 변경 0건.

### 체크리스트

- [x] **Spelix realm 진입** → Clients → Create
- [x] **client 신설** — `demo-dify-chat`
  - Client Type: OpenID Connect
  - Client authentication: OFF (public client)
  - **Direct access grants: ON** (ROPC 사용을 위해 필수)
  - Standard flow: ON (PKCE 콜백 흐름도 유지)
  - **Require PKCE: ON** + **PKCE Method: S256** (Capability config 단계에 토글 노출. 구버전 Keycloak에서는 Advanced 탭의 "Proof Key for Code Exchange Code Challenge Method" 드롭다운에 해당 — 본질 동일)![[Pasted image 20260528142129.png]]![[Pasted image 20260528141600.png]]![[Pasted image 20260528142157.png]]
- [x] **client role 생성** — `demo-dify-chat-user` (demo-dify-chat client → Roles → Create)
- [x] **데모 사용자 3명 신규 생성** — Users → Add user
  - Username: `demo-01`, `demo-02`, `demo-03`
  - Email Verified: ON (이메일 미사용)
  - 각 사용자 → Credentials 탭 → Set password → 임시 패스워드 입력 + **Temporary: OFF** (다음 로그인 시 비밀번호 변경 강제 비활성)
	  - password: 123 ![[Pasted image 20260528152600.png]]
- [x] **각 데모 사용자에 role 직접 부여** — Users → demo-XX → Role mapping → Assign role
  - Filter by clients → `demo-dify-chat` 선택 → `demo-dify-chat-user` 체크 → Assign
  - demo-01, demo-02, demo-03 각각 동일 작업![[Pasted image 20260528152724.png]]
- [x] **redirect_uri 등록** — demo-dify-chat client → Settings → Valid redirect URIs
  - `https://n8n159.spelix.co.kr/api/auth/callback` (배포)
  - `http://localhost:3000/api/auth/callback` (dev)![[Pasted image 20260528153242.png]]
- [x] **CORS 허용** — Settings → Web origins
  - `https://n8n159.spelix.co.kr`
  - `http://localhost:3000`![[Pasted image 20260528153248.png]]
- [x] **검증** (5/28 PowerShell 통과)
  - `demo-01`로 ROPC 토큰 발급 성공 (200 + access/refresh)
  - access_token 디코드 → `resource_access['demo-dify-chat'].roles: ["demo-dify-chat-user"]` 확인
  - issuer `http://192.168.10.194:8080/realms/Spelix` / azp `demo-dify-chat` / `email_verified: true` 모두 정합
  - `allowed-origins`에 배포 + dev 도메인 포함 (CORS 정상)
  - **트러블슈팅 기록**: 최초 `invalid_grant + "Account is not fully set up"` 발생 — Keycloak User Profile 정책상 First/Last name/Email 필수. demo-01/02/03 모두 채워야 통과 (`Demo / User01 / 01@demo.com` 등 가짜 값으로 OK)

### 산출물

- demo-dify-chat client (public + Direct Access Grants + PKCE)
- client role `demo-dify-chat-user` 생성
- 데모 사용자 3명 (`demo-01`, `demo-02`, `demo-03`) 신규 생성 + role 직접 부여
- 토큰 발급 + role 포함 + spx-agent onboarding 차단 검증 통과

---

## PR #2 산출물 (5/28 완료)

- **브랜치**: `keycloak-login`
- **커밋**: `d6b8a92` (publish 완료)
- **백엔드 신규 파일 4종**: `keycloak_auth.py` / `token.py` / `auth_decorator.py` / `routes/auth.py`
- **백엔드 수정**: `config.py` 환경변수 / `app.py` Blueprint 등록 + CORS / 기존 5 Blueprint 16개 라우트에 `@login_required`
- **프론트엔드 신규**: `lib/auth.ts` (`authFetch` + `useAuth`)
- **프론트엔드 수정**: 로그인 페이지 (Keycloak 연동) / `(main)/layout.tsx` (`useAuth`) / 헤더 (사용자명 + 로그아웃)
- **테스트**: curl 9 시나리오 + Next.js 프록시 5 시나리오 = **14건 전부 PASS**

### PR #2 트러블슈팅 기록

- **role 이름 mismatch** — 설계상 `demo-user`였으나 PR #1에서 실제 생성된 건 `demo-dify-chat-user`. PR #2 코드 작성 시 후자로 통일 (설계 노트도 일괄 교체)
- **`.env` 유실 사고** — `.gitignore`에 포함된 상태에서 새로 만들다 기존 DB URL / Dify API 키 / SMTP 유실. 복원 완료. 교훈: 기존 `.env`가 존재하는지 먼저 확인 후 추가 형태로 작업
- **좀비 Flask 프로세스** — `pkill -f "python app.py"`는 부모만 죽고 자식 프로세스 남을 수 있음. Windows에서는 `taskkill //F //IM python.exe`로 전체 정리 필요
- **Next.js 프록시 cookie 전달** — rewrite는 same-origin이라 fetch 기본값(`credentials: 'same-origin'`)으로도 cookie 자동 전송됨. `credentials: 'include'` 미지정이어도 동작 (단 명시 권장)

---

## 작업 단계 — PR #2+: web-ui 코드 통합

> 승랑님 6 커밋 (`f3e9dbe keycloak sso → c8cc19c → 0ff4434 → d10a5f9 → a687c1a → 3acfba4`) 패턴을 web-ui로 **경량 이식** (~320줄, spx-agent 1,000줄+ 대비 약 1/3).
>
> 소스 분석 상세: [[0. Inbox/2026-05-28 web-ui Keycloak Phase 1 — 핵심 흐름 분석]] / Phase 2 / Phase 3 / Phase 4
> 이식 계획 차이 분석: [[0. Inbox/2026-05-28 web-ui Keycloak 이식 계획 vs 설계 노트 차이 정리]]

### 인증 외 경계는 이 PR 범위 외

```
사용자 브라우저
    │ (Keycloak cookie 인증)  ← 이 PR 책임 범위
    ▼
web-ui Frontend (Next.js)
    │ (same-origin)
    ▼
web-ui Backend (Flask)
    │ (Dify API key — 기존 그대로, 이 PR 무관)
    ▼
Dify API → LLM
```

→ Backend가 LLM 직접 호출 안 함 (`services/dify_proxy.py`가 Dify `/chat-messages`만 호출). Keycloak 토큰을 LLM에 첨부하는 작업 **불필요**.

### 이식 매핑

```
spx-agent (참고)                    web-ui (이식 결과)
──────────────────                  ──────────────────
keycloak.py (405줄, 6 API)    →    backend/routes/auth.py (~150줄, 4 API)
passport.py (95줄)            →    backend/libs/keycloak_auth.py (~60줄)
token.py (237줄)              →    backend/libs/token.py (~80줄)
login.py (119줄)              →    backend/libs/auth_decorator.py (~30줄)
ext_login.py (179줄)          →    생략 (Flask-Login 미사용)
account_service.py            →    생략 (web-ui 백엔드에 accounts 테이블 없음)
wraps.py                      →    생략 (Keycloak 전용, 이중 모드 없음)
keycloak_admin.py             →    생략 (Admin API 미사용)
```

### 의존성 및 환경 변수

```bash
# 디렉토리/패키지 사전 작업
mkdir -p backend/libs && touch backend/libs/__init__.py

# requirements.txt에 추가
PyJWT
cryptography
# (requests는 이미 포함)
```

```env
KEYCLOAK_URL=http://192.168.10.194:8080
KEYCLOAK_INTERNAL_URL=http://192.168.10.194:8080
KEYCLOAK_REALM=Spelix
KEYCLOAK_ISSUER=http://192.168.10.194:8080/realms/Spelix   # 토큰 iss claim과 일치해야 함
KEYCLOAK_CLIENT_ID=demo-dify-chat
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30
COOKIE_SECURE=false           # dev=false / 배포=true. token.py가 Set-Cookie의 Secure 플래그 판정에 사용
FLASK_SECRET_KEY=<랜덤 생성>
```

> `COOKIE_SECURE` 환경변수 채택 이유: gunicorn 뒤에서 `request.is_secure`가 X-Forwarded-Proto 누락 시 False 반환 위험. 환경변수로 명시하면 환경 의존성 0. 추후 nginx 설정 명확해지면 `ProxyFix` 미들웨어로 전환 가능.

### 백엔드 (Flask) — 신규 파일 4개

| 파일 | 크기 | 책임 |
|---|---|---|
| `backend/libs/keycloak_auth.py` | ~60줄 | JWKS RS256 검증 (PyJWKClient 싱글턴 + 캐싱) + role 검증 함수 |
| `backend/libs/token.py` | ~80줄 | Cookie 2종 (access/refresh) set/extract/clear. Secure 플래그는 `COOKIE_SECURE` 환경변수 참조 |
| `backend/libs/auth_decorator.py` | ~30줄 | `@login_required` (token 추출 → 검증 → role 확인 → `g.current_user` 세팅) |
| `backend/routes/auth.py` | ~150줄 | 인증 API 4종 (아래) |

#### 인증 엔드포인트 4종

- [x] **`POST /api/auth/login-password`** — 주력. ID/PW 받아 Keycloak ROPC token 요청 → JWKS RS256 검증 → role 확인 → cookie 2종 설정 → `{result:success}` 응답
- [x] **`POST /api/auth/refresh`** — refresh_token cookie로 access 재발급 (DB 조회 없음, JWT sub 직접 사용)
- [x] **`POST /api/auth/logout`** — **옵션 B**: 백엔드가 Keycloak에 token revoke + cookie 2종 clear + `{result:success}` 응답 (redirect_url 반환 안 함)
- [x] **`GET /api/auth/me`** — 신규. cookie에서 token 추출 → 검증 → `{sub, name, email}` 반환 (프론트엔드 세션 확인용)

> PKCE 흐름(`login` + `callback`)과 `force-login`은 **이식하지 않음**. 데모 UX는 ROPC만 사용 + ROPC 환경에서 SSO 세션 미생성. 향후 외부 SSO 필요 시 별도 작업.

#### role 검증 패턴

```python
# auth_decorator.py
roles = decoded.get("resource_access", {}) \
               .get("demo-dify-chat", {}) \
               .get("roles", [])
if "demo-dify-chat-user" not in roles:
    return jsonify({"code": "forbidden", "message": "Access denied"}), 403
```

#### 계정 provisioning — **생략 확정**

- web-ui 백엔드 모델: `Room`, `RoomReservation`, `Schedule`만. accounts/users 테이블 없음
- spx-agent의 `AccountService.provision_default_workspace_for_keycloak` 이식 불필요
- 사용자 정보는 JWT claims에서 매 요청마다 직접 추출 (`sub`, `name`, `email`)
- 향후 "사용자별 데이터" 필요 시 비즈니스 테이블에 `created_by` 컬럼(JWT `sub` 저장) 추가로 처리. 별도 `accounts` 테이블 신설 불필요

#### Cookie 2종 (CSRF 초기 생략)

| Cookie | httpOnly | SameSite | 비고 |
|---|---|---|---|
| `access_token` | ✅ | Lax | JS에서 읽을 수 없음 |
| `refresh_token` | ✅ | Lax | JS에서 읽을 수 없음 |
| ~~`csrf_token`~~ | — | — | **초기 생략** — SameSite=Lax + httpOnly + 단일 도메인 조합으로 CSRF 방어 충분 |

> CSRF 토큰은 cross-origin POST가 가능한 정책 변경(다른 도메인 임베드 등) 발생 시 추가. 현재 구조에서는 marginal value 낮음.

### 기존 라우트에 인증 적용

- [x] `chat.py`, `document.py`, `files.py`, `rooms.py`, `schedules.py` 5개 Blueprint의 모든 라우트에 `@login_required` 데코레이터 추가 (총 **16개 라우트** 적용 완료)

### Flask app 설정 변경

- [x] `backend/app.py`: `auth_bp` 등록
- [x] CORS 설정: `origins=['http://localhost:3000', 'https://n8n159.spelix.co.kr']` + `supports_credentials=True`

### 프론트엔드 (Next.js)

- [x] **인증 헬퍼 신규** — `frontend/lib/auth.ts`
  - **`authFetch(url, options)` 래퍼**: 모든 보호 API 호출의 단일 진입점
    - `credentials: 'include'` 자동 첨부
    - 401 응답 시 → `POST /api/auth/refresh` 시도 → 성공 시 원 요청 재시도, 실패 시 `/login` redirect
    - refresh 동시 호출 방지(in-flight promise 공유) — 60분 만료 직후 여러 컴포넌트가 동시 호출 시 race 회피
  - **`useAuth()` 훅**: `authFetch('/flask-api/api/auth/me')` 호출 → `{sub, name, email}` / `null` 반환
    - layout + Header 양쪽에서 재사용
- [x] **로그인 페이지 수정** — `frontend/app/login/page.tsx`
  - 필드명: `email` → `username` (label "아이디")
  - submit: `POST /flask-api/api/auth/login-password` (`credentials: 'include'` 필수, `authFetch` 미사용 — 로그인 자체는 401 intercept 대상 아님)
  - 성공 → `/` 이동 / 실패 → 에러 메시지 (`code: 'authentication_failed'` 분기)
  - sessionStorage 제거
- [x] **라우트 보호 수정** — `frontend/app/(main)/layout.tsx`
  - `useAuth()` 훅 사용 → 비로그인이면 `/login` redirect
- [x] **헤더에 사용자명 + 로그아웃 버튼** — Header 컴포넌트
  - `useAuth()` 훅으로 사용자명 표시
  - 로그아웃: `POST /flask-api/api/auth/logout` (`credentials: 'include'`) → `/login` 이동
- [ ] **기존 API 호출도 `authFetch`로 통일** (선택, PR #2 후속) — chat / document / files / rooms / schedules 컴포넌트의 fetch 호출을 점진적으로 `authFetch`로 교체. 그래야 60분 후에도 끊김 없는 UX 보장 → **Next.js rewrite 프록시가 same-origin이라 기존 fetch도 cookie 자동 전송 (default `credentials: 'same-origin'`)되어 동작은 함. 다만 401 시 자동 refresh 없으므로 점진적 교체 권장**

### 검증 시나리오

**로컬 환경 (5/28 완료)**:
- [x] 데모 계정 `demo-01`로 로그인 → 홈 진입 성공, Keycloak 도메인 노출 안 됨
- [x] 비-DEMO 계정으로 로그인 시도 → 403 응답, 로그인 화면 유지
- [x] 액세스 토큰 만료 후 보호 API 호출 → `authFetch`가 자동 refresh → 정상 응답 (강제 로그아웃 없음)
- [x] refresh_token까지 만료된 상태로 보호 API 호출 → `/login` redirect
- [x] 로그아웃 클릭 → web-ui 로그인 화면 복귀 (에러 페이지 표시 없음)
- [x] 로그아웃 직후 동일 브라우저로 다시 로그인 → 정상 동작
- [x] `@login_required` 미적용 라우트 0건 검증 (5개 Blueprint 전수 — 16개 라우트 모두 적용)
- [x] cookie 없는 상태로 보호 라우트 호출 → 401 응답
- [x] curl 9 시나리오 (직접 Flask) + Next.js 프록시 5 시나리오 = **14건 전부 PASS**

**배포 환경 (5/29 예정)**:
- [ ] 배포 환경에서 cookie `Secure` 플래그 ON 확인 (`COOKIE_SECURE=true` 설정 효과 검증)
- [ ] 배포 도메인 `https://n8n159.spelix.co.kr/`에서 위 9개 시나리오 재실행

---

## 리스크 및 주의 사항

| 항목 | 내용 | 대응 |
|---|---|---|
| ROPC 보안 트레이드오프 | 백엔드가 사용자 비밀번호 직접 처리 | 데모 환경 한정 사용, 외부 OAuth 클라이언트 미공개로 위험 통제 |
| KEYCLOAK_SESSION cookie 잔존 | 로그아웃 옵션 B 사용 시 브라우저에 Keycloak 세션 cookie 남음 | ROPC만 사용하므로 사용자 브라우저가 Keycloak 도메인에 접근할 일 없어 실질 영향 없음 |
| 라이브러리 충돌 | React 19 + Next 16 환경, 외부 OIDC 라이브러리 미사용 | 충돌 위험 거의 없음 — P2 분석 결과 |
| 승랑님 패턴 부정합 | spx-agent는 백엔드 Flask, web-ui도 백엔드 Flask | 구조 유사 — 큰 이식 부담 없음 |
| issuer URL 정합 | Keycloak issuer가 `http://192.168.10.194:8080/realms/Spelix` (HTTP + 사내 IP). 토큰 iss claim 검증 시 정확히 일치 필요 | `KEYCLOAK_ISSUER` 환경변수로 명시 |
| 한글 인코딩 | 한글 부서명/이름이 토큰 claims에 포함될 가능성 | UTF-8 통일, cookie 인코딩 확인 |
| 토큰 만료 시 API CPU 행 | spx-agent 사례 (gevent worker 환경) | web-ui는 gunicorn **sync worker** 사용 → threading.Lock 정상 동작, 본 패턴 발생 위험 낮음. [[4. 지식노트/Keycloak - 토큰 만료 시 API CPU 100% 행 패턴]] 참조 |
| **토큰 만료 시 강제 로그아웃 UX** | access 60분 만료 후 단순 redirect 시 외부 고객 데모 중단 | `authFetch` 래퍼가 401 intercept → refresh 자동 시도 → 재시도. refresh도 실패하면 그때만 `/login` |
| **Cookie Secure 플래그** | gunicorn 뒤에서 `request.is_secure`가 X-Forwarded-Proto 누락 시 False 반환 | `COOKIE_SECURE` 환경변수로 명시 통제 (`token.py`가 이 값 참조) |
| **refresh token lifespan 정합 미해결** | Keycloak realm refresh lifespan 실측 **2시간** (5/28 토큰 `exp - iat` 검증). 설계 `REFRESH_TOKEN_EXPIRE_DAYS=30`과 큰 차이 → 외부 고객 데모 2시간 초과 시 강제 로그아웃 | **5/29 결정 대기** — demo-dify-chat client → Advanced 탭에서 토큰 lifespan override vs 운영 규칙(2시간 내 끝내기) |

## 참고 자료

- [[0. Inbox/2026-05-28 web-ui Keycloak Phase 1 — 핵심 흐름 분석]] — keycloak.py / passport.py / token.py 처리 순서 + 이식 시 주의사항
- [[0. Inbox/2026-05-28 web-ui Keycloak Phase 2 — 보호 계층 분석]] — login.py / ext_login.py + CSRF 흐름 + 경량화 의사코드
- [[0. Inbox/2026-05-28 web-ui Keycloak Phase 3 — 프론트엔드 연동 분석]] — mail-and-password-auth.tsx / common.ts + credentials: 'include' 필수
- [[0. Inbox/2026-05-28 web-ui Keycloak Phase 4 — 선택적 참고 분석]] — account_service / wraps / configs 생략 판단 매트릭스
- [[0. Inbox/2026-05-28 web-ui Keycloak 이식 계획 vs 설계 노트 차이 정리]] — Step 1-10 이식 계획 + 차이 6건 결정
- [[0. Inbox/2026-05-26 web-ui Keycloak 통합 사전 분석 위임 (P2)]] — P2 분석 위임 프롬프트 (인풋 / 작업 원칙 / 후속 14항)
- [[1. Daily/2026-05-26]] — P2 분석 결과 (톱 4 확정, 통합 전략 결정, 승랑님 6 커밋 식별)
- [[1. Daily/2026-05-28]] — AAI 주간 보고 회의록 (데모 대상 외부 / DEMO 계정 / "dify 계정이어도 ㄱㅊ") + 소스 직접 확인 (ROPC 발견) + 인프라 사전 검증
- `C:\Users\Administrator\Projects\spx-agent\.claude\docs\references\keycloak-sync.md` — Keycloak 동기화 운영 절차 + 매칭 우선순위 + 트러블슈팅
- `C:\Users\Administrator\Projects\spx-agent\api\controllers\console\auth\keycloak.py` — 이식 기준 백엔드 구현 (405 lines, 6 엔드포인트). 특히 line 202-286 `login-password` (ROPC) + line 362-404 `logout`
- `C:\Users\Administrator\Projects\spx-agent\api\libs\passport.py` — JWKS RS256 검증 헬퍼
- [[3. 프로젝트/web-ui/로그인 인증 설계]] — 3/30 초기 모듈 선택 단계 노트 (본 노트의 선행)
- [[4. 지식노트/Keycloak - 토큰 만료 시 API CPU 100% 행 패턴]]

## 관련 노트

- [[3. 프로젝트/web-ui/리뉴얼 설계 v2]]
- [[SPX-Agent 중앙 관리 대시보드]]
