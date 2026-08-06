---
tags: [dify, 개발, Next.js, 프론트엔드]
date: 2026-04-29
---
# Dify - 프론트엔드 라우트 및 페이지 추가 방법

## 라우팅 구조

**Next.js 15+ App Router** (폴더 기반 라우팅)

```
web/app/
├── (commonLayout)/        ← 인증된 사용자 공간 (사이드바/헤더)
│   ├── layout.tsx         ← RoleRouteGuard + Header 포함
│   ├── apps/page.tsx
│   ├── datasets/page.tsx
│   ├── tools/page.tsx
│   └── explore/page.tsx
├── (shareLayout)/         ← 공개 공유 페이지
├── account/               ← 계정 관련
└── ...
```

폴더 경로 = URL. 예: `(commonLayout)/apps/page.tsx` → `/apps`

## 새 페이지 추가 절차

### Step 1: 폴더 + page.tsx 생성

```
web/app/(commonLayout)/admin/dashboard/page.tsx
```

```tsx
const DashboardPage = () => {
  return <div>Dashboard</div>
}
export default DashboardPage
```

→ URL: `/admin/dashboard`

### Step 2: 동적 라우트 (선택)

```
web/app/(commonLayout)/admin/dashboard/[id]/page.tsx
```

`useParams()`로 `params.id` 접근

### Step 3: 레이아웃 상속

부모 `(commonLayout)/layout.tsx`가 자동 적용 → RoleRouteGuard, Header, AppContextProvider 상속

## 네비게이션/사이드바 메뉴 추가

### 헤더 Nav 추가

1. **Nav 컴포넌트 생성**: `web/app/components/header/admin-nav/index.tsx`
   - 기존 패턴: `app-nav`, `dataset-nav`, `tools-nav`, `explore-nav`
   - `Nav` 컴포넌트로 감싸기

2. **Header에 등록**: `web/app/components/header/index.tsx`
   ```tsx
   <AdminNav className={navClassName} />
   ```

### 설정 모달에 탭 추가

1. **상수 추가**: `web/app/components/header/account-setting/constants.ts`
   ```tsx
   export const ACCOUNT_SETTING_TAB = {
     ...
     DASHBOARD: 'dashboard',
   }
   ```

2. **탭 아이템 정의**: `account-setting/index.tsx`
   ```tsx
   {
     key: ACCOUNT_SETTING_TAB.DASHBOARD,
     name: '대시보드',
     icon: <DashboardIcon />,
     activeIcon: <DashboardIcon />,
   }
   ```

3. **렌더 로직 추가**:
   ```tsx
   {activeMenu === ACCOUNT_SETTING_TAB.DASHBOARD && <DashboardPage />}
   ```

## 인증/가드

| 계층 | 위치 | 역할 |
|------|------|------|
| **AppContextProvider** | `context/app-context-provider.tsx` | 프로필/워크스페이스 자동 로드, 로딩 중 렌더링 차단 |
| **RoleRouteGuard** | `(commonLayout)/role-route-guard.tsx` | DatasetOperator 역할이면 특정 라우트 차단 → `/datasets` 리다이렉트 |
| **공유 인증** | `(shareLayout)/components/authenticated-layout.tsx` | 권한 없으면 403 표시 |

## 주요 파일 경로

| 항목 | 경로 |
|------|------|
| 인증 필수 레이아웃 | `web/app/(commonLayout)/layout.tsx` |
| 라우트 가드 | `web/app/(commonLayout)/role-route-guard.tsx` |
| App 컨텍스트 | `web/context/app-context-provider.tsx` |
| 메뉴/헤더 | `web/app/components/header/index.tsx` |
| 설정 모달 | `web/app/components/header/account-setting/index.tsx` |
| 설정 상수 | `web/app/components/header/account-setting/constants.ts` |

## 관련 노트

- [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[4. 지식노트/spx-agent - 프로젝트 폴더 구조.md]]
