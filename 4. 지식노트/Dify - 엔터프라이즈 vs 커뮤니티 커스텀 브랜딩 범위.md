---
tags: [dify, 개발, AI-Agent]
date: 2026-04-27
---
# Dify - 엔터프라이즈 vs 커뮤니티 커스텀 브랜딩 범위

## 핵심
- Dify의 **화이트라벨링**(로그인 로고, 콘솔 헤더, 탭 제목, 파비콘 교체)은 **엔터프라이즈 라이선스에서만** 가능
- 커뮤니티(셀프호스트)에서는 `CAN_REPLACE_LOGO=true` 설정 시 **WebApp의 "Powered by Dify" 영역만** 커스텀 가능
- 엔터프라이즈 기능은 `ENTERPRISE_ENABLED=true` + 유효한 라이선스가 필수이며, 라이선스 만료/비활성 시 강제 로그아웃됨

## 상세

### 엔터프라이즈 전용 — 시스템 전체 브랜딩 (`branding`)

`ENTERPRISE_ENABLED=true`일 때 라이선스 서버에서 설정값을 받아 시스템 전체에 적용된다.

| 항목 | 설명 | 적용 위치 |
|------|------|-----------|
| `login_page_logo` | 로그인/회원가입/비밀번호 재설정 페이지 로고 | `signin/_header.tsx` 등 인증 페이지 전체 |
| `workspace_logo` | 콘솔 상단 헤더, 계정 정보, 공유앱 사이드바 로고 | `header/index.tsx`, `account-about/index.tsx` |
| `application_title` | 브라우저 탭 제목 (`"Dify"` → 커스텀) | `use-document-title.ts` |
| `favicon` | 브라우저 파비콘 교체 | `use-document-title.ts` |

**백엔드 흐름:**
```
Enterprise API → enterprise_info["Branding"] → BrandingModel → 프론트엔드 systemFeatures.branding
```

**프론트엔드 분기 로직:**
```tsx
// branding 꺼짐 → Dify 기본 로고
// branding 켜짐 → 커스텀 로고
{systemFeatures.branding.enabled && systemFeatures.branding.login_page_logo
  ? <img src={systemFeatures.branding.login_page_logo} />
  : <DifyLogo />}
```

> 이 설정은 **코드로 직접 설정 불가** — 라이선스 계약 후 Enterprise API에서 내려오는 값이다.

### 커뮤니티 버전 — WebApp "Powered by" 영역만

`.env`에 `CAN_REPLACE_LOGO=true`를 설정하면 활성화된다.

```python
# api/configs/enterprise/__init__.py
CAN_REPLACE_LOGO: bool = Field(
    description="Allow customization of the enterprise logo.",
    default=False,
)
```

활성화 시 워크스페이스 설정에 **Custom** 탭이 나타나며 두 가지를 할 수 있다:

| 기능 | 설명 |
|------|------|
| `remove_webapp_brand` | WebApp 하단 `"POWERED BY Dify"` 문구 자체를 숨기기 (토글) |
| `replace_webapp_logo` | `"POWERED BY"` 옆 Dify 로고를 커스텀 로고로 교체 (SVG/PNG, 5MB 이하) |

**WebApp 로고 표시 우선순위** (3단계):
1. 엔터프라이즈 `branding.workspace_logo` (최우선)
2. 워크스페이스별 `custom_config.replace_webapp_logo` (CAN_REPLACE_LOGO)
3. 기본 Dify 로고 (폴백)

### 비교 요약

| 항목 | 커뮤니티 (`CAN_REPLACE_LOGO=true`) | 엔터프라이즈 |
|------|------|------|
| 로그인 페이지 로고 | Dify 고정 | 커스텀 가능 |
| 콘솔 헤더 로고 | Dify 고정 | 커스텀 가능 |
| 브라우저 탭 제목 | `"Dify"` 고정 | 커스텀 가능 |
| 파비콘 | Dify 아이콘 고정 | 커스텀 가능 |
| WebApp "Powered by" 숨기기 | 가능 | 가능 |
| WebApp 로고 교체 | 워크스페이스별 가능 | 시스템 로고 우선 적용 |

### 기타 엔터프라이즈 전용 기능 (브랜딩 외)

- **SSO** (SAML/OIDC/OAuth2) 강제 로그인
- **WebApp 인증/접근 제어** (public/private/sso_verified 모드)
- **워크스페이스 권한 정책** (멤버 제한, 초대/이전 허용 여부)
- **플러그인 설치 범위 제어** (official_only 등)
- **Enterprise 텔레메트리** (OpenTelemetry 기반 트레이싱/메트릭)
- **라이선스 검증 미들웨어** (만료 시 강제 로그아웃)

### 주요 소스 경로

- 설정: `api/configs/enterprise/__init__.py`
- 기능 플래그 조합: `api/services/feature_service.py`
- Enterprise 전용 데코레이터: `api/controllers/console/wraps.py` (`only_edition_enterprise`)
- WebApp 사이트 정보: `api/controllers/web/site.py`
- 프론트엔드 브랜딩 분기: `web/hooks/use-document-title.ts`, `web/app/signin/_header.tsx`
- WebApp 커스텀 UI: `web/app/components/custom/custom-web-app-brand/`
- 시스템 기능 타입: `web/types/feature.ts`

## 관련 노트
- [[4. 지식노트/Dify - 워크스페이스·통계·토큰 기존 코드 구조.md]]
- [[4. 지식노트/spx-agent - 프로젝트 폴더 구조.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[2. 회의록/0427 AAI 데일리 스크럼.md]]
