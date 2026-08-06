---
tags: [인증, JWT, 프론트엔드, 패턴]
date: 2026-05-28
---

# 웹 인증 - authFetch 래퍼 패턴 (401 intercept + auto refresh)

> httpOnly cookie + access/refresh 토큰 조합에서 access가 만료되면 사용자를 곧장 `/login`으로 보내지 말 것. **refresh token이 살아 있으면 자동 갱신 후 원 요청 재시도**가 올바른 UX. fetch 래퍼 하나로 모든 보호 API에 일괄 적용.

## 문제 시나리오 — 단순 redirect의 함정

access 60분 / refresh 30일 설정에서 단순 redirect 시:

```
사용자가 61분 후 페이지 이동
→ layout이 /api/auth/me 호출
→ access_token 만료 → 401
→ /login으로 redirect
→ refresh_token은 살아 있는데도 강제 재로그인 ❌
```

외부 고객 데모 / 장시간 작업 화면에서 치명적.

## 해결 — authFetch 래퍼

모든 보호 API 호출의 단일 진입점:

```typescript
// lib/auth.ts
let inflightRefresh: Promise<boolean> | null = null

async function refreshToken(): Promise<boolean> {
  // in-flight promise 공유 — 60분 만료 직후 여러 컴포넌트가
  // 동시에 401 받아도 refresh는 한 번만 호출
  if (inflightRefresh) return inflightRefresh
  
  inflightRefresh = fetch('/api/auth/refresh', {
    method: 'POST',
    credentials: 'include',
  }).then(r => r.ok).finally(() => {
    inflightRefresh = null
  })
  
  return inflightRefresh
}

export async function authFetch(url: string, options: RequestInit = {}): Promise<Response> {
  let res = await fetch(url, { ...options, credentials: 'include' })
  
  if (res.status === 401) {
    const refreshed = await refreshToken()
    if (refreshed) {
      // refresh 성공 → 원 요청 재시도 (새 access_token cookie 자동 첨부)
      res = await fetch(url, { ...options, credentials: 'include' })
    } else {
      // refresh도 실패 → 그제서야 /login
      window.location.href = '/login'
    }
  }
  return res
}
```

## 핵심 설계 요소

### 1. `credentials: 'include'` 자동 첨부

httpOnly cookie 기반 인증의 필수 조건. 매 호출에서 일일이 지정하면 누락 위험 → 래퍼가 강제.

### 2. 401 intercept → refresh 시도 → 원 요청 재시도

표준 흐름. 핵심은 **재시도가 한 번뿐**이라는 것 — 재시도도 401이면 무한 루프 회피.

### 3. in-flight promise 공유 (race 회피)

만료 직후 여러 컴포넌트가 동시 호출하는 시나리오:
```
시각 T: access 만료
시각 T+1ms: layout이 /me 호출 → 401 → refresh 시작
시각 T+2ms: Header가 /me 호출 → 401 → 또 refresh 시작 (race!)
시각 T+3ms: 다른 API 호출 → 401 → 또 refresh 시작
```

각각 별도 refresh 보내면:
- Keycloak이 refresh_token rotation 사용 시 첫 번째만 통과, 나머지 invalid
- 또는 race condition으로 cookie 덮어쓰기 충돌

`inflightRefresh` promise 공유로 **첫 호출만 실제 refresh 수행, 나머지는 그 결과 대기**.

### 4. 로그인 API는 authFetch 미사용

로그인 자체(`/login-password`)는 401 intercept 대상 아님 — 잘못된 패스워드도 401인데 그걸로 refresh 트리거하면 안 됨. 로그인은 일반 `fetch`로.

## useAuth 훅과 결합

```typescript
export function useAuth() {
  const [user, setUser] = useState<User | null | undefined>(undefined)
  
  useEffect(() => {
    authFetch('/api/auth/me')
      .then(res => res.ok ? res.json() : null)
      .then(setUser)
      .catch(() => setUser(null))
  }, [])
  
  return user  // undefined=loading, null=비로그인, User=로그인
}
```

layout + Header 양쪽에서 호출해도 in-flight 공유로 안전.

## 점진적 적용

기존 fetch 호출을 한 번에 다 바꾸기 부담스러우면:

1. **신규 보호 API 호출 = `authFetch` 의무화** (린트 룰)
2. 기존 호출은 점진 교체
3. same-origin 환경이면 기존 fetch도 cookie 자동 전송되므로 동작은 함 (단 401 자동 refresh 안 됨)

## 관련 노트

- [[4. 지식노트/웹 인증 흐름 - JWT, 쿠키, 세션]]
- [[4. 지식노트/인증 아키텍처 용어 - BaaS, IAM, SSO, 미들웨어]]
- [[4. 지식노트/Keycloak - 토큰 만료 시 API CPU 100% 행 패턴]]
- 적용 사례: [[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]] PR #2 프론트엔드 `lib/auth.ts`
