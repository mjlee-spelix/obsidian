---
tags: [프로젝트, dify, AI-Agent, HDD]
type: harness/guide
date: 2026-04-29
last_updated: 2026-05-13
---
# Architecture — 파일 위치 + 등록 방법

> 새 파일을 어디에 만들고, 어디를 수정해야 하는지 판단하기 위한 문서.

## 1. 프로젝트 레이아웃 (신규 코드 위치)

```
api/
├── controllers/console/dashboard/  ← 대시보드 API 컨트롤러 (신규)
│   ├── __init__.py
│   └── dashboard.py                ← KPI, dept-objects, model-tokens, dept-activity, drill-through 등
├── services/admin/                 ← 대시보드 서비스 레이어 (폴더명 admin 유지, 향후 dashboard/로 통일 검토)
│   ├── __init__.py
│   ├── dashboard_kpi_service.py
│   ├── dashboard_dept_objects_service.py
│   ├── dashboard_model_tokens_service.py
│   ├── dashboard_dept_activity_service.py
│   └── dashboard_drill_*_service.py (objects/users/calls/errors)
├── models/                         ← SQLAlchemy 모델
│   └── rbac.py                     ← RBAC 테이블 모델 (회사 표준 명명 — `spx_` 접두사)
└── tests/
    ├── unit_tests/services/admin/                          ← 서비스 단위 테스트
    └── integration_tests/controllers/console/dashboard/    ← API 통합 테스트

web/
├── app/(commonLayout)/dashboard/   ← 대시보드 페이지 (신규, /dashboard 라우트)
│   └── page.tsx                    ← 페이지 헤더 + 자식 컴포넌트 마운트
├── app/components/admin/           ← 대시보드 전용 컴포넌트 (폴더명 admin 유지, 향후 dashboard/로 통일 검토)
│   ├── dashboard-controls/         ← 페이지 헤더 슬롯 컨트롤 (기간/새로고침)
│   ├── kpi-section/                ← KPI 카드 4종
│   ├── dept-objects-chart/
│   ├── model-tokens-chart/
│   ├── dept-activity-table/
│   ├── drill-charts/               ← Phase 2 drill-through 차트 (클릭 비활성 — 차트만 표시)
│   └── drill-tables/               ← Phase 2 drill-through 표
├── app/components/header/
│   ├── dashboard-nav/              ← 헤더 톱 네비 첫 번째 메뉴 (신규, ExploreNav 미러)
│   │   └── index.tsx
│   ├── index.tsx                   ← Header에 DashboardNav 추가 (수정)
│   └── account-setting/
│       ├── constants.ts            ← DASHBOARD 탭 상수 제거 (수정)
│       └── index.tsx               ← 사이드바 메뉴 + dashboard-page 렌더 제거 (수정)
└── service/
    ├── use-admin-dashboard.ts      ← API 훅 (신규)
    └── use-admin-drill.ts          ← drill-through 11종 훅 (신규)
```

> **마운트 환경**: 대시보드는 **`/dashboard` 톱레벨 라우트**에 마운트 (`web/app/(commonLayout)/dashboard/page.tsx`).
> - 사용자 범위: 로그인 사용자 전체
> - 로그인 시 디폴트 페이지: `/dashboard` (`DEFAULT_POST_LOGIN_PATH` 상수)
> - 자체 `<h1>` 페이지 헤더 작성, URL query string으로 상태 관리, ESC 자유 활용
> - `dashboard-controls`는 페이지 헤더 슬롯에 배치
> - 비로그인 가드는 `(commonLayout)`의 `AppInitializer`가 자동 적용 (→ `/signin`)

## 2. 백엔드 — 엔드포인트 등록 절차

### 아키텍처 계층

```
Flask App
  └─ Blueprint 등록 (api/extensions/ext_blueprints.py)  ← 이미 등록됨
       └─ Namespace (api/controllers/console/__init__.py)
            └─ Controller (api/controllers/console/admin/*.py)
                 └─ Service (api/services/admin/*.py)
```

### 등록 2단계 (Blueprint는 이미 있음)

**① 컨트롤러 파일 생성**

```python
# api/controllers/console/admin/dashboard.py
from controllers.console import console_ns
from flask_restful import Resource

@console_ns.route("/admin/dashboard/kpi")
class DashboardKpiApi(Resource):
    method_decorators = [
        account_initialization_required,
        login_required,
        setup_required,
    ]
    def get(self):
        return DashboardKpiService.get_kpi(...)
```

**② import 등록** — `api/controllers/console/__init__.py` 하단에 한 줄:

```python
from .admin import dashboard  # 이 import만으로 라우트 자동 등록
```

> Blueprint(`console_bp`, prefix `/console/api`)는 `ext_blueprints.py`에 이미 등록됨.
> 새 Blueprint 추가 불필요.

### 3개 API 레이어 (우리는 Console만 사용)

| 레이어 | URL Prefix | 인증 | 용도 |
|--------|-----------|------|------|
| **Console API** | `/console/api/` | `@login_required` (JWT) | **우리가 쓸 곳** |
| Service API | `/v1/` | `@validate_app_token` | 외부 개발자 |
| Inner API | `/inner/api/` | `X-Inner-Api-Key` | 내부 서비스 간 |

### 우리 엔드포인트 경로 (2026-05-13 admin prefix 폐기)

| 엔드포인트 | 메서드 | 용도 |
|-----------|--------|------|
| `/console/api/dashboard/kpi` | GET | KPI 카드 4종 (총 오브젝트 / 부서별 채택 앱 수 / API 호출 / 앱별 통계) |
| `/console/api/dashboard/dept-objects` | GET | 부서별 오브젝트 |
| `/console/api/dashboard/model-tokens` | GET | 모델별 토큰 |
| `/console/api/dashboard/dept-activity` | GET | 부서별 활동 |
| `/console/api/dashboard/drill/{metric}/{chart}` | GET | drill-through 11종 |

> CSV 내보내기는 정적 대시보드에서 제거됨. 차트 드로어는 보류(2026-05-13).

## 3. 백엔드 — 서비스 레이어

```
Controller (요청 파싱, 응답 직렬화)
    ↓
Service (비즈니스 로직, DB 쿼리)
    ↓
Model (SQLAlchemy ORM)
```

- Controller: 얇게. 파라미터 파싱 → Service 호출 → jsonify.
- Service: `@classmethod`로 정의. SQLAlchemy 쿼리 직접 작성.
- 외부 HTTP 호출 필요 시: `httpx` + `tenacity` 재시도 패턴 (`billing_service.py` 참고).

## 4. 백엔드 — 모델 위치

| 모델 | 파일 | 참고 |
|------|------|------|
| `App`, `AppMode` | `api/models/model.py` | 기존 Dify |
| `Message` | `api/models/model.py` | `message_tokens`, `answer_tokens` |
| `WorkflowRun` | `api/models/workflow.py` | `total_tokens` |
| `Account`, `Tenant` | `api/models/account.py` | 기존 Dify |
| `Dataset`, `Document` | `api/models/dataset.py` | 기존 Dify |
| `Tag`, `TagBinding` | `api/models/model.py` | type="app" / "knowledge" |
| RBAC 5종 (`Department`, `DepartmentMember`, `ResourceOwnership`, `ResourcePermission`, `RbacAuditLog`) | `api/models/rbac.py` | 회사 표준 명명 — `spx_` 접두사 (`spx_departments` / `spx_department_members` / `spx_resource_ownership` / `spx_resource_permissions` / `spx_rbac_audit_logs`). `references/rbac-schema.md` 참조 |
| `AuditEvent` | (audit DB, 별도 스키마) | 마트 ETL 입력 단일 SoT. `references/audit-details-spec.md` 참조 |
| 마트 객체 4종 (`VAuditEnriched`, `VResourceOwnershipEnriched`, `MvKpiCallsDaily`, `MvModelTokensDaily`) | `api/models/mart.py` (신규) | View/MView 매핑. `hdd/design.md § 2.5.4` DDL 참조 |

> **마트 인터페이스 의무화 (2026-05-18)**: 백엔드 service는 위 마트 객체 4종(`spx_mv_audit_enriched` / `spx_mv_kpi_calls_daily` / `spx_mv_model_tokens_daily` / `spx_v_resource_ownership_enriched`)만 SELECT. 다음은 모두 금지 — collector P0 + enriched view에 이미 흡수됐기 때문에 우회 시 H-DASH-01/03 방어 누락.
> - ❌ raw 테이블 직접 SELECT (`messages` / `workflow_runs` / `workflow_node_executions`)
> - ❌ `spx_audit_events.details->>` raw JSON 추출
> - ❌ `_d` 접미사 Generated Column 직접 참조 (enriched view alias만 사용)
>
> **View/MView 모델 신설 패턴**: `api/models/mart.py`에 `__table_args__ = {'info': {'is_view': True}}` 마커로 Alembic autogen 제외. DDL은 Prisma migration이 소유 (`dify-audit/prisma/audit/migrations/`). 권한은 `ALTER OWNER TO audit_writer` + `GRANT SELECT` 적용됨 (5/15 Layer 2 작업 시 박힘).
>
> 위반 게이트: `hdd/specs/tasks/data-mart.md § 6단계 30번` grep 검증 4종 (PR 직전 0 hit 필수).

## 5. 프론트엔드 — 라우팅 (2026-05-13 변경: 톱레벨 라우트)

### 마운트 방식

대시보드는 **`/dashboard` 톱레벨 라우트**에 마운트 (`web/app/(commonLayout)/dashboard/page.tsx`).

- 사용자 범위: **로그인 사용자 전체**
- 로그인 시 디폴트 페이지: `/dashboard`
- 헤더 톱 네비 첫 번째 메뉴 = `DashboardNav`

### 라우트 페이지 추가 패턴

```
web/app/(commonLayout)/dashboard/
└── page.tsx                ← 페이지 헤더 + 자식 컴포넌트 마운트

web/app/components/header/
├── dashboard-nav/          ← 헤더 네비 진입점 (신규, ExploreNav 미러)
│   └── index.tsx
└── index.tsx               ← <DashboardNav />를 첫 번째 위치에 추가 (수정)
```

**수정·신규 위치:**
1. `web/app/(commonLayout)/dashboard/page.tsx` — 페이지 신설. 자체 `<h1>` + URL query string 상태 관리 + `dashboard-controls`를 페이지 헤더 슬롯에 배치
2. `web/app/components/header/dashboard-nav/` — 폴더 신설. ExploreNav 패턴 미러. 아이콘 `RiDashboardFill` / `RiDashboardLine` (`@remixicon/react`)
3. `web/app/components/header/index.tsx` — `<DashboardNav />` 첫 번째 위치 추가 + 5개 메뉴 균등 가운데 배치 (구현 시 확인)
4. **로그인 후 디폴트 redirect**: `DEFAULT_POST_LOGIN_PATH = '/dashboard'` 상수 도입 후 7군데 일괄 교체 (`web/app/signin/utils/post-login-redirect.ts` 등)
5. **i18n**: `web/i18n/ko-KR/common.ts` + `en-US/common.ts`에 "대시보드" / "Dashboard" 키 추가
6. **설정 모달 DASHBOARD 탭 제거**: `web/app/components/header/account-setting/{constants.ts, index.tsx}`에서 기존 대시보드 탭 항목 + 렌더 분기 제거. `dashboard-page/` 폴더 삭제

### 인터랙션 정책

- **좌하 차트 클릭**: 동작 없음 (차트만 표시 — 5/13 차트 드로어 보류로 트리거 제거)
- **표 행 클릭**: KPI 4번(앱별 통계)만 → 같은 탭으로 Dify 모니터링 페이지(`/app/{appId}/overview`). KPI 1·2·3 표 클릭 없음
- **KPI 카드 클릭**: drill-through 펼침 (Phase 2)
- **ESC 키**: 자유 활용 (drill-through 해제 등)

## 6. 프론트엔드 — 데이터 페칭

```
web/service/use-admin-dashboard.ts   ← API 훅 파일 (신규)
```

TanStack Query 패턴 (2026-05-13 admin 세그먼트 폐기):
```typescript
export const useAdminDashboardKpi = (params: DateRangeParams) => {
  return useQuery({
    queryKey: ['dashboard', 'kpi', params],
    queryFn: () => get('/dashboard/kpi', { params }),
    staleTime: 5 * 60 * 1000,  // 5분
  })
}
```

페이지 헤더의 새로고침 버튼은 `queryClient.invalidateQueries({ queryKey: ['dashboard'] })`로 일괄 무효화.

## 7. 프론트엔드 — 인증 가드 계층

| 계층 | 위치 | 역할 |
|------|------|------|
| AppContextProvider | `web/context/app-context-provider.tsx` | 프로필/워크스페이스 자동 로드 |
| RoleRouteGuard | `(commonLayout)/role-route-guard.tsx` | DatasetOperator 차단 |
| 설정 모달 | `account-setting/index.tsx` | 모달 내부 탭 전환 |

## 8. 기존 코드 참조 파일

| 목적 | 참고할 파일 |
|------|-----------|
| DB 집계 쿼리 | `api/controllers/console/app/statistic.py` |
| 어드민 API | `api/controllers/console/admin.py` (505줄) |
| API Key 관리 | `api/controllers/console/apikey.py` |
| Service 패턴 | `api/services/billing_service.py` |
| 차트 컴포넌트 | `web/app/components/app/overview/app-chart.tsx` |
| TanStack Query 훅 | `web/service/use-apps.ts` |
| 설정 모달 탭 | `web/app/components/header/account-setting/members-page/index.tsx` |
| 태그 관리 | `web/app/components/base/tag-management/` |

## 9. 개발 환경 표준 (2026-05-06 정리)

> 신규 개발자/환경 셋업 시 이 절차를 따르지 않으면 vitest/ESLint hook 부팅 불가 또는 lint 사후 위반 발생.
> 관련 결함: H-ENV-03 (Node drift), CAND-husky-activation-drift (defect-catalog.md)

### 9.1 환경 표준 매트릭스

| 항목 | 표준 값 | 출처 |
|------|---------|------|
| **호스트 Node** | **22** | `.nvmrc=22` (회사 표준) |
| **패키지 매니저** | **pnpm@10.33.0** | `package.json` `packageManager` 필드 |
| **활성화 도구** | **corepack** | Node 16.10+ 기본 포함, 글로벌 pnpm 설치 금지 |
| **Python** | 3.11+ | Dify 요구사항 (`datetime.UTC` alias 사용 가능) |
| **Pre-commit hook 경로** | `.vite-hooks/_` | `git config core.hooksPath` 설정값. husky 폴더 rename 패턴 |
| **Hook 활성화 트리거** | `pnpm install` prepare script | 자동으로 `.vite-hooks/_/` 폴더 + wrapper 생성 |

### 9.2 신규 셋업 절차 (체크리스트)

```bash
# 1. Node 22 설치 + 활성화
nvm install 22
nvm use 22
node --version    # v22.x.x 확인 필수

# 2. corepack 활성화 (한 번만, Windows는 관리자 권한 필요할 수 있음)
corepack enable

# 3. 레포 clone 후 의존성 설치 (corepack이 packageManager 필드 자동 매칭)
cd <repo-root>
pnpm install      # ← 이 단계에서 hook도 자동 활성화

# 4. 환경 검증
git config --get core.hooksPath   # .vite-hooks/_ 출력 → hook 활성화 확인
ls .vite-hooks/_                   # pre-commit, commit-msg 등 wrapper 존재 확인

# 5. 테스트 부팅 검증
cd web
pnpm vitest run app/components/admin/dashboard-controls   # 6/6 통과 + 1.67s
```

**검증 포인트** — 다음 중 하나라도 실패하면 환경 미완:
- `node --version` → v22 미만 → H-ENV-03 발동
- `git config --get core.hooksPath` → 빈 출력 → hook 미활성, lint 사후 발견 위험
- `pnpm vitest run ...` → "module 'node:fs/promises' does not provide an export named 'glob'" → Node 22 미만

### 9.3 Pre-commit hook 동작 메커니즘

```
git commit 명령
    ↓
git이 core.hooksPath 확인 → .vite-hooks/_/pre-commit (wrapper) 호출
    ↓
wrapper가 .vite-hooks/_/h (helper) 경유로 .vite-hooks/pre-commit (사용자 정의) 실행
    ↓
사용자 정의 스크립트가 ruff (api/) + ESLint (web/) 실행
    ↓
위반 발견 → commit 차단
```

**구조 분리**:
- `.vite-hooks/pre-commit` — 사용자 정의 스크립트, **git에 추적됨** (모든 사람 받음)
- `.vite-hooks/_/` — 자동 생성 wrapper, `.gitignore`로 **미추적** (개인별)

→ `.vite-hooks/` 폴더 자체는 추적되지만, 활성화 폴더 `_/` 는 각 개발자가 `pnpm install` 실행해야 생성됨. **= 개발자별 활성화 시점 차이로 lint 사후 발견 가능** (CAND-husky-activation-drift).

### 9.4 글로벌 패키지 매니저 설치 금지 — 이유

```bash
# ❌ 금지
npm install -g pnpm
npm install -g pnpm@9   # 회사 표준 10.33.0과 다른 버전

# ✅ 권장
corepack enable          # packageManager 필드 자동 매칭
```

글로벌 설치 시 발생 문제:
- 개발자별 pnpm 버전 차이 → `lockfile` 충돌
- nvm Node 버전 전환 후 글로벌 패키지 사라짐 → 매번 재설치
- 회사 표준 강제력 없음

corepack은 **`packageManager` 필드를 따라 자동으로 정확한 버전 fetch**. 모든 개발자가 동일.

### 9.5 환경 검증 명령 모음

```bash
# 한 줄 진단
node --version && pnpm --version && git config --get core.hooksPath && ls .vite-hooks/_/

# 정상 출력 예시
# v22.x.x
# 10.33.0
# .vite-hooks/_
# .gitignore  h  pre-commit  commit-msg  ...
```

상세 학습 노트:
- [[4. 지식노트/Node - corepack과 패키지 매니저 버전 통일]]
- [[4. 지식노트/husky - .husky 폴더 패턴과 install 시점]]
- [[4. 지식노트/Git - core.hooksPath와 Hook 위치 추적]]
