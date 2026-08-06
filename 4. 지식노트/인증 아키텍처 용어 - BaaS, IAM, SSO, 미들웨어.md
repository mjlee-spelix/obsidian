---
tags: [지식, 인증, 아키텍처, baas, iam, sso, 미들웨어]
date: 2026-03-30
---
# 인증 아키텍처 용어 - BaaS, IAM, SSO, 미들웨어

## 핵심
- **BaaS**: 백엔드를 직접 안 만들고 서비스로 갖다 쓰는 것 (Supabase, Firebase)
- **IAM**: 기업용 사용자/권한 관리 시스템 (Keycloak)
- **SSO**: 한 번 로그인으로 여러 서비스 이용
- **미들웨어**: 요청이 페이지에 도달하기 전에 가로채서 검사하는 관문

## 상세

### BaaS (Backend as a Service)

백엔드 기능(DB, 인증, 파일 저장 등)을 API로 제공하는 클라우드 서비스.

```
직접 구축:  프론트 → 내가 만든 서버 → 내가 관리하는 DB
BaaS 사용:  프론트 → Supabase API → Supabase가 관리하는 DB
```

- **Supabase**: 오픈소스 BaaS. PostgreSQL DB + Auth + Storage + Realtime을 하나로 제공
- **Firebase**: Google의 BaaS. Firestore DB + Auth + Hosting 등
- 서버 코드를 거의 안 짜고 프론트에서 직접 DB 조회, 인증 처리 가능
- 빠른 프로토타입, 소규모 프로젝트에 적합

### IAM (Identity and Access Management)

"누가(Identity) 무엇을(Access) 할 수 있는가(Management)"를 관리하는 시스템.

```
IAM의 세 가지 질문:
1. 이 사람이 누구인가? → 인증 (Authentication)
2. 이 사람이 무엇을 할 수 있는가? → 인가 (Authorization)
3. 이 사람의 역할은 무엇인가? → 역할 관리 (Role Management)
```

- **Keycloak**: 오픈소스 IAM 서버. 별도 서버로 설치해서 운영
- 기업에서 "관리자는 전체 메뉴, 일반 사용자는 일부 메뉴만" 같은 역할 기반 접근제어에 사용
- LDAP/Active Directory 연동 → 회사 기존 계정 시스템과 통합 가능

**인증(Authentication) vs 인가(Authorization)**:

| 구분 | 인증 (AuthN) | 인가 (AuthZ) |
|------|-------------|-------------|
| 질문 | "너 누구야?" | "너 이거 해도 돼?" |
| 예시 | 로그인 (이메일+비밀번호) | 관리자 페이지 접근 권한 체크 |
| 시점 | 먼저 수행 | 인증 이후에 수행 |

### SSO (Single Sign-On)

한 번 로그인하면 여러 연결된 서비스에 추가 로그인 없이 접근할 수 있는 방식.

```
SSO 없이:
  Gmail 로그인 → 비밀번호 입력
  YouTube 로그인 → 비밀번호 또 입력
  Drive 로그인 → 비밀번호 또 입력

SSO 사용:
  Google 계정 로그인 → Gmail ✅ YouTube ✅ Drive ✅ (자동)
```

- Keycloak이 SSO를 제공하는 대표적인 도구
- 사내에 여러 시스템(ERP, 그룹웨어, 메일 등)이 있을 때, 하나의 Keycloak 로그인으로 전부 이용
- **OIDC (OpenID Connect)** 프로토콜이 SSO의 표준 구현 방식

### 미들웨어 (Middleware)

요청(request)이 실제 페이지에 도달하기 전에 **중간에서 가로채는** 코드.

```
사용자 요청 → [미들웨어] → 페이지
              │
              ├─ 토큰 있음 → 통과 → 페이지 렌더링
              └─ 토큰 없음 → 차단 → /login으로 리다이렉트
```

Next.js에서는 `middleware.ts` 파일이 이 역할을 함:

```typescript
// middleware.ts (개념 코드)
export function middleware(request) {
  const token = request.cookies.get('token');

  if (!token) {
    // 토큰 없으면 로그인 페이지로 보냄
    return NextResponse.redirect('/login');
  }

  // 토큰 있으면 원래 페이지로 통과
  return NextResponse.next();
}
```

- 모든 라우트에 대해 자동 실행되므로, 페이지마다 인증 체크 코드를 넣을 필요 없음
- 인증 외에도 로깅, 언어 감지, A/B 테스트 등에 활용

### OIDC (OpenID Connect)

OAuth 2.0 위에 만들어진 인증 프로토콜. "이 사람이 누구인지"를 표준화된 방식으로 확인.

```
Keycloak 로그인 흐름 (OIDC):

1. 사용자가 "로그인" 클릭
2. → Keycloak 로그인 페이지로 리다이렉트
3. Keycloak에서 이메일/비밀번호 입력
4. → 우리 사이트 /callback?code=xyz 로 리다이렉트
5. 서버가 code를 Keycloak에 보내서 토큰으로 교환
6. 로그인 완료
```

- Supabase의 소셜 로그인(Google, GitHub)도 내부적으로 OIDC/OAuth를 사용
- Keycloak은 OIDC **제공자(Provider)**, next-auth는 OIDC **클라이언트**

## 관련 노트
- [[웹 인증 흐름 - JWT, 쿠키, 세션]]
- [[개발 도구 용어 - SDK, 패키지, Auth 라이브러리]]
- [[3. 프로젝트/web-ui/로그인 인증 설계]]
