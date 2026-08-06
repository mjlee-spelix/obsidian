---
tags: [프로젝트, web-ui, keycloak, 분석, phase-3]
date: 2026-05-28
related:
  - "[[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]]"
  - "[[0. Inbox/2026-05-28 web-ui Keycloak Phase 1 — 핵심 흐름 분석]]"
  - "[[0. Inbox/2026-05-28 web-ui Keycloak Phase 2 — 보호 계층 분석]]"
---

# Phase 3 — 프론트엔드 연동 분석

> "프론트엔드가 어떤 형식으로 호출하고 응답을 처리하는가"
> 분석 대상: `mail-and-password-auth.tsx` → `common.ts`

---

## 1. 로그인 폼 (`mail-and-password-auth.tsx` — 151줄)

### submit 핸들러 핵심 로직

```typescript
const handleUsernamePasswordLogin = async () => {
  // ① 빈값 검증
  if (!username?.trim()) { toast.error(...); return; }
  if (!password?.trim()) { toast.error(...); return; }

  // ② API 호출
  setIsLoading(true);
  const res = await login({
    url: '/keycloak/login-password',
    body: { username, password },
  });

  // ③ 성공 처리
  if (res.result === 'success') {
    if (res?.data?.access_token) {
      setWebAppAccessToken(res.data.access_token);  // localStorage에 저장
    }
    router.replace('/dashboard');  // 홈으로 이동
  }

  // ④ 에러 처리 (catch)
  catch (error) {
    const err = error as ResponseError;
    if (err?.code === 'authentication_failed')
      toast.error('아이디 또는 비밀번호가 잘못되었습니다');
    else
      toast.error(err?.message || '로그인 실패');
  }
};
```

### web-ui와의 차이점

| 항목 | spx-agent 프론트 | web-ui 현재 |
|------|-----------------|------------|
| 프레임워크 | Next.js (spx-agent 자체) | Next.js 16.1.6 (App Router) |
| API 호출 | `login()` from `service/common.ts` | 직접 fetch 또는 신규 헬퍼 |
| 로그인 URL | `/keycloak/login-password` | `/api/auth/login-password` (신규) |
| 성공 후 이동 | `/dashboard` | `/` (홈) |
| 에러 표시 | toast (spx-agent 컴포넌트) | 인라인 에러 메시지 (기존 UI) |
| 토큰 저장 | `setWebAppAccessToken` (localStorage) | **불필요** — httpOnly cookie 방식 |
| SSO 핸드오프 | `safeReturnTo()` 크로스도메인 | **불필요** — 단일 도메인 |

---

## 2. login() API 함수 (`common.ts:49-51`)

### 함수 시그니처

```typescript
type LoginSuccess = {
  result: 'success'
  data?: { access_token?: string }
}
type LoginFail = {
  result: 'fail'
  data: string
  code: string
  message: string
}
type LoginResponse = LoginSuccess | LoginFail

export const login = ({ url, body }: { 
  url: string, 
  body: Record<string, any> 
}): Promise<LoginResponse> => {
  return post<LoginResponse>(url, { body })
}
```

### web-ui 이식

web-ui에는 spx-agent의 `service/base.ts` (post/get/put/del 래퍼)가 없으므로, **직접 fetch 호출**로 구현:

```typescript
// web-ui용 간소화 버전
async function loginApi(username: string, password: string) {
  const res = await fetch('/flask-api/api/auth/login-password', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',  // ← cookie 송수신 필수!
    body: JSON.stringify({ username, password }),
  });
  
  if (!res.ok) {
    const err = await res.json();
    throw err;
  }
  return res.json();
}
```

### credentials: 'include' 필수

httpOnly cookie 기반 인증에서는 `credentials: 'include'`가 반드시 있어야 한다:
- 로그인 응답의 `Set-Cookie`가 브라우저에 저장됨
- 이후 API 요청 시 cookie가 자동 첨부됨

---

## 3. web-ui 현재 로그인 페이지 분석 (`frontend/app/login/page.tsx`)

### 현재 구현 (stub)

```typescript
// 현재: sessionStorage 통과형
sessionStorage.setItem('user', JSON.stringify({ email }));
router.push('/');
```

### 수정 계획

```
현재:
  submit → sessionStorage 저장 → / 이동

변경 후:
  submit → POST /flask-api/api/auth/login-password
         → 성공: / 이동 (cookie는 브라우저가 자동 관리)
         → 실패: 에러 메시지 표시
```

### 변경 필요 사항

1. **email → username 변경**: Keycloak은 username 기반 (email도 가능하지만 필드명은 username)
2. **fetch 호출 추가**: `POST /flask-api/api/auth/login-password`
3. **sessionStorage 제거**: cookie 기반으로 전환
4. **에러 처리 추가**: 401/403 응답에 따른 메시지 분기

---

## 4. 라우트 보호 (`frontend/app/(main)/layout.tsx`)

### 현재 구현 (stub)

```typescript
useEffect(() => {
  const user = sessionStorage.getItem('user');
  if (!user) {
    router.replace('/login');
  }
}, [router]);
```

### 수정 계획

세션 확인을 **백엔드 API 호출**로 변경:

```typescript
// 방법 A: /api/auth/me 엔드포인트 호출 (권장)
useEffect(() => {
  fetch('/flask-api/api/auth/me', { credentials: 'include' })
    .then(res => {
      if (!res.ok) router.replace('/login');
      else return res.json().then(data => setUser(data));
    })
    .catch(() => router.replace('/login'));
}, []);

// 방법 B: 단순 cookie 존재 확인 (빠르지만 보안 약함)
// csrf_token cookie는 httpOnly가 아니므로 JS에서 확인 가능
useEffect(() => {
  const hasCsrf = document.cookie.includes('csrf_token=');
  if (!hasCsrf) router.replace('/login');
}, []);
```

**방법 A 권장** — 서버에서 토큰 유효성까지 확인. 다만 백엔드에 `/api/auth/me` 엔드포인트 추가 필요.

---

## 5. 로그아웃 버튼

### 현재 상태

Header 컴포넌트에 로그아웃 기능 없음.

### 추가 필요

```typescript
const handleLogout = async () => {
  await fetch('/flask-api/api/auth/logout', {
    method: 'POST',
    credentials: 'include',
  });
  router.replace('/login');
};
```

---

## 6. Next.js 프록시 설정 확인

### 현재 `next.config.ts`

```typescript
rewrites() {
  return [{
    source: '/flask-api/:path*',
    destination: `${FLASK_BACKEND}/:path*`,
  }];
}
```

→ `/flask-api/api/auth/login-password` → Flask `POST /api/auth/login-password`로 프록시됨.

### 쿠키 전달 이슈

Next.js rewrites는 same-origin이므로 cookie가 자연스럽게 전달됨. **추가 설정 불필요**.

단, Flask CORS 설정에 `supports_credentials=True` 추가 필요:

```python
# backend/app.py
CORS(app, 
     origins=['http://localhost:3000', 'https://n8n159.spelix.co.kr'],
     supports_credentials=True)
```

---

## Phase 3 요약 — 프론트엔드 수정 목록

| 파일 | 변경 내용 | 난이도 |
|------|----------|--------|
| `app/login/page.tsx` | sessionStorage → fetch API 호출, email→username, 에러 처리 | 낮음 |
| `app/(main)/layout.tsx` | sessionStorage 체크 → `/api/auth/me` 호출 | 낮음 |
| `components/Header.tsx` | 로그아웃 버튼 + 사용자명 표시 추가 | 낮음 |
| `next.config.ts` | 변경 없음 (기존 프록시 그대로 사용) | — |
