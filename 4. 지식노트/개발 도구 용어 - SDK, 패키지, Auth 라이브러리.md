---
tags: [지식, 개발도구, sdk, 패키지, auth]
date: 2026-03-30
---
# 개발 도구 용어 - SDK, 패키지, Auth 라이브러리

## 핵심
- **패키지**: 남이 만든 코드를 `npm install`로 가져다 쓰는 것
- **SDK**: 특정 서비스를 쉽게 쓸 수 있도록 만든 패키지 묶음
- **Auth 라이브러리**: 인증 기능을 대신 처리해주는 SDK/패키지

## 상세

### 패키지 (Package)

재사용 가능하도록 묶어놓은 코드 단위. npm(Node.js의 패키지 매니저)으로 설치.

```bash
npm install react-markdown    # 마크다운 렌더링 패키지
npm install winston            # 로깅 패키지
npm install @supabase/ssr      # Supabase SSR 인증 패키지
```

- `package.json`의 `dependencies`에 기록됨
- `node_modules/` 폴더에 실제 코드가 다운로드됨
- 다른 언어에서는 라이브러리, 모듈, gem(Ruby), pip 패키지(Python) 등으로 부름

### SDK (Software Development Kit)

특정 서비스/플랫폼을 사용하기 위한 **도구 모음**. 패키지보다 넓은 개념.

```
패키지: 하나의 기능을 하는 코드 묶음
  예) react-markdown → 마크다운을 HTML로 변환

SDK: 특정 서비스의 여러 기능을 묶은 패키지 세트
  예) @supabase/supabase-js → DB 조회, 인증, 파일 업로드 등 Supabase 전체 기능
```

SDK는 보통 이런 것들을 포함:
- API를 호출하는 함수들 (직접 fetch 안 써도 됨)
- 타입 정의 (TypeScript 자동완성)
- 에러 처리, 재시도 로직
- 문서와 예제 코드

```typescript
// SDK 없이 (직접 API 호출):
const res = await fetch('https://xxx.supabase.co/auth/v1/token', {
  method: 'POST',
  headers: { 'apikey': '...', 'Content-Type': 'application/json' },
  body: JSON.stringify({ email, password, grant_type: 'password' })
});

// SDK 사용:
const { data, error } = await supabase.auth.signInWithPassword({ email, password });
```

### Auth 라이브러리 (인증 관련 패키지들)

인증 모듈을 선택하면 설치하게 되는 패키지들:

**Supabase 선택 시:**

| 패키지 | 역할 |
|--------|------|
| `@supabase/supabase-js` | Supabase 핵심 SDK (DB, Auth, Storage 등 전체) |
| `@supabase/ssr` | Next.js 같은 SSR 환경에서 쿠키 기반 세션 관리 |

```typescript
// 사용 예시
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)
await supabase.auth.signInWithPassword({ email, password })  // 로그인
await supabase.auth.signOut()                                  // 로그아웃
const { data: { user } } = await supabase.auth.getUser()      // 현재 사용자
```

**Keycloak 선택 시:**

| 패키지 | 역할 |
|--------|------|
| `next-auth` (Auth.js) | 다양한 인증 제공자를 통합하는 Next.js용 인증 라이브러리 |
| `@auth/core` | next-auth의 코어 엔진 |

```typescript
// 사용 예시
import { useSession, signIn, signOut } from 'next-auth/react'

const { data: session } = useSession()       // 현재 세션
await signIn('keycloak')                     // Keycloak 로그인 (리다이렉트)
await signOut()                               // 로그아웃
```

### 비교: 개발자 경험 차이

```
Supabase:
  로그인 폼 제출 → supabase.auth.signInWithPassword() → 토큰 자동 관리 → 끝
  (로그인 UI를 직접 만들고, SDK가 나머지 처리)

Keycloak + next-auth:
  로그인 버튼 클릭 → signIn('keycloak') → Keycloak 페이지로 이동 → 로그인 → 자동 복귀
  (로그인 UI는 Keycloak이 제공, next-auth가 토큰 교환/세션 관리)
```

## 관련 노트
- [[웹 인증 흐름 - JWT, 쿠키, 세션]]
- [[인증 아키텍처 용어 - BaaS, IAM, SSO, 미들웨어]]
- [[3. 프로젝트/web-ui/로그인 인증 설계]]
