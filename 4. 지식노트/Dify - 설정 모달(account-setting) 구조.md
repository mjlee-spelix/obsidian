---
tags: [dify, 개발, AI-Agent]
date: 2026-04-28
---
# Dify - 설정 모달(account-setting) 구조

## 핵심
- Dify 설정은 별도 라우트(`/settings`)가 **아님** — 헤더 프로필에서 열리는 **모달 다이얼로그** 안에 탭으로 구성
- 탭 추가는 **`constants.ts`에 상수 추가 + `index.tsx`에 메뉴 항목·렌더링 추가** 2곳이면 완료
- 중앙 관리 대시보드는 이 설정 모달의 워크스페이스 그룹에 새 탭으로 추가할 예정

## 상세

### 설정 모달 진입 경로

```
헤더 프로필 드롭다운 → "설정" 클릭
    → AccountSetting 모달 열림 (URL 파라미터로 상태 관리)
    → 좌측 사이드바에 탭 목록 표시
```

모달 상태는 **nuqs** (URL 파라미터 기반 상태 관리)로 관리됨. `useAccountSettingModal()` 훅으로 열기/닫기/탭 전환.

### 파일 구조

```
web/app/components/header/account-setting/
├── index.tsx              ← 메인: 사이드바 메뉴 + 탭 렌더링
├── constants.ts           ← 탭 상수 정의 (ACCOUNT_SETTING_TAB)
├── menu-dialog.tsx        ← 모달 래퍼 컴포넌트
├── model-provider-page/   ← 모델 제공자 탭
│   ├── index.tsx
│   ├── hooks.ts
│   ├── declarations.ts
│   ├── atoms.ts
│   ├── model-auth/
│   ├── model-modal/
│   ├── model-selector/
│   └── provider-added-card/
├── members-page/          ← 멤버 탭
│   └── index.tsx
├── data-source-page-new/  ← 데이터 소스 탭
│   └── index.tsx
├── api-based-extension-page/ ← API 확장 탭
│   └── index.tsx
└── language-page/         ← 언어 설정 탭
    └── index.tsx
```

### 현재 탭 목록

**워크스페이스 그룹** (dataset operator 역할에게는 숨김):

| 탭 상수 | 표시 이름 | 아이콘 | 조건 |
|--------|----------|--------|------|
| `PROVIDER` | 모델 제공자 | `i-ri-brain-2-line` | 항상 |
| `MEMBERS` | 멤버 | `i-ri-group-2-line` | 항상 |
| `BILLING` | 빌링 | `i-ri-money-dollar-circle-line` | `enableBilling` 시 |
| `DATA_SOURCE` | 데이터 소스 | `i-ri-database-2-line` | 항상 |
| `API_BASED_EXTENSION` | API 확장 | `i-ri-puzzle-2-line` | 항상 |
| `CUSTOM` | 커스텀 브랜딩 | `i-ri-color-filter-line` | `enableReplaceWebAppLogo` 시 |

**계정 그룹**:

| 탭 상수 | 표시 이름 | 아이콘 |
|--------|----------|--------|
| `LANGUAGE` | 언어 | `i-ri-translate-2` |

### 새 탭 추가 방법 (3단계)

#### ① `constants.ts` — 탭 상수 추가

```typescript
ACCOUNT_SETTING_TAB = {
  PROVIDER: 'provider',
  MEMBERS: 'members',
  DASHBOARD: 'dashboard',   // ← 추가
  // ...
}
```

#### ② `index.tsx` — 사이드바 메뉴 + 렌더링

**사이드바 메뉴 항목 추가** (워크스페이스 그룹 내):
```tsx
{
  key: ACCOUNT_SETTING_TAB.DASHBOARD,
  name: '중앙 관리 대시보드',
  icon: 'i-ri-dashboard-line',  // Remix Icon
}
```

**탭 렌더링 조건 추가**:
```tsx
{activeMenu === ACCOUNT_SETTING_TAB.DASHBOARD && <DashboardPage />}
```

#### ③ 탭 페이지 컴포넌트 생성

`account-setting/dashboard-page/index.tsx` 생성. 다른 탭들(`members-page/index.tsx` 등)과 동일한 패턴.

### 모달 열기 (다른 곳에서 대시보드 탭 직접 열기)

```typescript
import { useAccountSettingModal } from '...'
import { ACCOUNT_SETTING_TAB } from '...'

// 대시보드 탭으로 바로 열기
setShowAccountSettingModal({ payload: ACCOUNT_SETTING_TAB.DASHBOARD })
```

### 모달 컨텍스트 관리

| 파일 | 역할 |
|------|------|
| `web/context/modal-context-provider.tsx` | 모달 열기/닫기 상태, URL 파라미터 연동 |
| `web/app/components/header/account-dropdown/index.tsx` | 드롭다운에서 설정 모달 여는 진입점 |
| `web/app/components/header/index.tsx` | 헤더에서 빌링 탭 직접 여는 진입점 |

### 기존 앱 통계 UI와의 차이

| | 앱 통계 (overview) | 중앙 관리 대시보드 (설정 모달) |
|--|-------------------|------------------------|
| **위치** | `/app/[appId]/overview/` | 설정 모달 내 탭 |
| **범위** | 앱 1개의 일별 통계 | 워크스페이스 전체·부서별 |
| **데이터** | `messages`, `workflow_runs` (단일 앱) | + `object_ownership`, `departments` (RBAC) |
| **라우팅** | Next.js 페이지 라우트 | 모달 내부 탭 전환 (URL 파라미터) |

## 관련 노트
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[4. 지식노트/Dify - 새 API 엔드포인트 등록 방법.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 소스 분석 현황.md]]
