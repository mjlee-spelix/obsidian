# SPX-Agent 중앙 관리 대시보드

> 이 파일은 에이전트의 **진입점 지도**입니다.
> 깊은 내용은 포인터로 연결된 문서에서 확인하세요.
>
> **세션 시작 시**: `SESSION_HISTORY.md`를 가장 먼저 읽고 진행 상태를 파악하세요.
> **세션 종료 시**: 의미 있는 변경(결정·패턴·규칙)이 있으면 `SESSION_HISTORY.md`의 "변경 이력"에 한두 줄 추가하세요.

## 프로젝트 개요

- **설명**: Dify 셀프호스트 워크스페이스의 중앙 운영 대시보드 (로그인 사용자 전체 공개 — 2026-05-13 변경, 기존 "어드민 전용" 폐기)
- **기술 스택**: Flask, SQLAlchemy, Next.js 14, PostgreSQL, Redis
- **베이스 코드**: Dify (오픈소스) — 기존 코드 무수정 원칙, 신규 모듈만 추가
- **범위**: KPI 카드 4종(총 오브젝트 / 총 이용 앱 수 / 총 앱 호출량 / 인기 호출 앱), 부서별 오브젝트 차트, 모델별 토큰 차트, 부서별 활동 테이블
- **페이지 소개 문구** (`/dashboard` 헤더 좌측, i18n key `common.menus.dashboardDescription`):
  - 한국어: `앱, 지식, 도구의 사용 현황과 부서별 활동을 한눈에 확인합니다.`
  - 영어: `Monitor app, knowledge, and tool usage with department activity at a glance.`

## 빠른 시작

```bash
# 환경 활성화 (한 번만)
nvm use 22                         # 회사 표준 Node (.nvmrc=22)
corepack enable                    # pnpm 10.33.0 자동 매칭 (글로벌 설치 금지)

# 백엔드
cd api && poetry install && flask run

# 프론트엔드
cd web && pnpm install && pnpm dev

# 테스트
cd api && pytest tests/
cd web && pnpm vitest run app/components/admin/<component>   # 범위 좁혀서 (1~2초)

# Docker 빌드 (Windows — 깨진 symlink 자동 정리 후 빌드)
.\scripts\spx-build.ps1            # 전체
.\scripts\spx-build.ps1 api        # 특정 서비스
# 단독 정리만: .\scripts\spx-clean-broken-symlinks.ps1
# ※ Windows PowerShell 5.1 / PowerShell 7 모두 동작 (pwsh 명시 불필요)
# 관련 결함: H-ENV-01, H-ENV-02, H-ENV-03 (defect-catalog.md)
# 도커 패턴 학습: [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]]
```

## 디렉토리 구조 (2026-05-13 갱신)

```
api/
├── controllers/console/dashboard/  ← 대시보드 API 엔드포인트 (신규, /console/api/dashboard/ — 5/13 admin/ → dashboard/ 이동)
├── services/admin/                 ← 대시보드 비즈니스 로직 (신규, 폴더명 admin/ 유지 또는 dashboard/로 통일 검토)
├── models/                         ← SQLAlchemy 모델 (RBAC 테이블 — `spx_` 접두사, 회사 표준)
└── core/app/                       ← Dify 기존 앱 실행 로직 (수정 금지)

web/
├── app/(commonLayout)/dashboard/   ← 대시보드 페이지 (신규, /dashboard 라우트 — 5/13 admin/dashboard/ → dashboard/ 변경)
├── app/components/header/dashboard-nav/  ← 헤더 톱 네비 메뉴 첫 번째 (신규, ExploreNav 미러)
├── app/components/admin/           ← 대시보드 컴포넌트 (폴더명 유지 또는 dashboard/로 통일 검토)
└── app/components/header/account-setting/  ← 설정 모달 (수정 — 기존 DASHBOARD 탭 제거 검토)
```

> **5/13 라우트/사이드바 결정 사항**:
> - URL = `/dashboard` (admin prefix 폐기, 톱레벨)
> - 헤더 메뉴 = 대시보드 / 탐색 / 스튜디오 / 지식 / 도구 (5개, 대시보드 첫 번째)
> - API 경로 = `/console/api/dashboard/` (UI와 통일)
> - 사용자 범위 = 로그인 사용자 전체 (관리자 전용 폐기)
> - dataset_operator 권한도 접근 허용 (RoleRouteGuard 추가 X)
> - i18n: 한국어 "대시보드" / 영어 "Dashboard"
> - 아이콘: `RiDashboardFill` / `RiDashboardLine` (`@remixicon/react`)

## 도메인 지식 (포인터)

| 문서 | 역할 | 언제 읽나 |
|------|------|----------|
| `SESSION_HISTORY.md` | 진행 상태 + 핵심 결정 + 변경 이력 | 세션 시작 시 **항상 먼저** |
| `architecture.md` | 파일 위치 + 등록 방법 | 새 파일 만들 때 **필독** |
| `conventions.md` | 코드 스타일 + 테스트 패턴 | 코드 작성 시 |
| `hdd/defect-catalog.md` | 도메인 함정 패턴 | 구현 시작 전 **필독** |
| `hdd/design.md` | 변경 영향 규칙 + 의존 관계 | 설계 변경 시 |
| `hdd/specs/requirements/{component}.md` | 컴포넌트별 요구사항 + Harness 방어 | 해당 컴포넌트 구현 시작 시 **필독** |
| `hdd/specs/design/{component}.md` | API + 쿼리 + 컴포넌트 설계 | 해당 컴포넌트 구현 시 |
| `hdd/specs/tasks/{component}.md` | 구현 순서 체크리스트 | 해당 컴포넌트 구현 시 |
| `hdd/delegation-standard.md` | AI 위임 시 박을 가드 5종 (drift / 자가 검증 / idempotent / 컨테이너 / 한글 인코딩) | **위임 PROMPT.md 작성 시 필독** |

**현재 컴포넌트 목록 (`{component}` 자리에 들어가는 이름)**:
- **Phase 1 (정적)**: `dashboard-controls`, `kpi-cards`, `dept-objects`, `model-tokens`, `dept-activity`
- **Phase 2 (동적 인터랙션)**: `kpi-drill-through`
- **🔒 보류**:
  - `chart-drawer` (2026-05-13 이사님 결정 — H-DASH-20 참조. spec 3파일 본문 보존, 재검토 시 base 활용)
  - `context-bar` (2026-05-22 보류 결정 — H-DASH-21 참조. `page.tsx`에서 렌더링만 비활성, 컴포넌트 코드(`web/app/components/admin/context-bar/`)는 보존. spec 3파일 본문 base 보존, 재도입 시 PM 결정 동반)

> **마운트 환경 (2026-05-13 변경)**: 대시보드는 `/dashboard` 톱레벨 라우트로 마운트됨 (`web/app/(commonLayout)/dashboard/page.tsx`). ~~설정 모달 탭~~ 폐기. 따라서:
> - 컴포넌트가 **자체 `<h1>`/페이지 헤더 작성 가능** (모달 제약 해제)
> - **상태 관리는 URL query string + React state** — Phase 2 drill-through deep link 자연스러움 (모달 제약 해제)
> - `dashboard-controls`는 페이지 헤더 슬롯에 배치 (모달 헤더 우측 슬롯 패턴 폐기)
> - **ESC 키 자체 핸들링 가능** — 모달 환경이 아니라 ESC는 자유롭게 활용 (drill-through 해제 / 드로어 닫기 등)
> - 모든 대시보드 페이지는 `(commonLayout)` 하위라 `AppInitializer` 가드(비로그인 → `/signin`) 자동 적용
> - 좌하 차트 클릭 인터랙션 없음 — 차트만 표시 (5/13 차트 드로어 보류로 좌하 클릭 트리거 제거)
> - 표 행 클릭은 KPI 4번(인기 호출 앱)만 → Dify 모니터링 페이지 이동(`/app/{appId}/overview`). KPI 1·2·3 표 클릭은 동작 없음

> **개발 환경 셋업 주의** (2026-05-06 정리): 신규 환경/개발자 셋업 시 다음 조건 필수. 미충족 시 vitest/ESLint hook 부팅 불가 또는 lint 사후 위반 발생.
> - **호스트 Node 22** (회사 표준 `.nvmrc=22`). Node 20 이하면 `vinext@0.0.40`의 `fs.glob` 호출 실패로 vitest + ESLint hook + vite.config 로드 모두 부팅 불가 (H-ENV-03).
> - **`corepack enable`** 후 `pnpm install` — `package.json`의 `packageManager: pnpm@10.33.0` 자동 매칭. **글로벌 pnpm 설치 금지** (drift 위험).
> - **Pre-commit hook**: `.vite-hooks/` (husky 폴더 rename 패턴). `pnpm install`의 prepare script로 자동 활성화. 검증: `git config --get core.hooksPath` → `.vite-hooks/_` 출력.
> - **Hook 미활성 환경에서 commit 시 위험** — 다른 개발자의 lint 위반이 master에 사후 진입 가능. CI lint job이 유일한 강제 게이트.
> - 상세 절차: `architecture.md § 9 — 개발 환경 표준`. 학습 노트: [[4. 지식노트/Node - corepack과 패키지 매니저 버전 통일]], [[4. 지식노트/husky - .husky 폴더 패턴과 install 시점]], [[4. 지식노트/Git - core.hooksPath와 Hook 위치 추적]]

> 각 spec frontmatter의 `design_image:` / `reference_image:` / `current_implementation_image:` 필드에 이미지 경로가 있으면 **반드시 함께 읽어 1:1 대조**할 것.
>
> **이미지 폴더 구조** (`.claude/images/`):
> - `설계/` — 화면 설계 원본 (예: `(화면 설계) 대시보드.png`)
> - `Dify/` — Dify 참조 이미지 (예: `(Dify) 모니터링 - 챗봇.png`)
> - `구현/` — 우리가 만든 화면 캡처 (예: `(화면 구현) KPI 지표 2차 0430.png`)
>
> spec frontmatter 경로 형식: `images/{설계|Dify|구현}/...` — `.claude/images/...`로 해석.
| `hdd/quality-criteria.md` | 자기 평가 기준 | 구현 완료 후 검증 시 |
| `references/dify-db-schema.md` | Dify 테이블 스키마 | 쿼리 작성 시 |
| `references/dify-app-modes.md` | AppMode별 동작 규칙 | 토큰/통계 쿼리 작성 시 |
| `references/rbac-schema.md` | RBAC 테이블 DDL (회사 표준 — `spx_` 접두사) | RBAC 관련 쿼리 작성 시 |
| `references/audit-details-spec.md` | audit_events details 필드 가용성 매트릭스 | 마트 ETL 쿼리 작성 시 |
| `references/objects-charts-feasibility.md` | 오브젝트 차트 3종 RBAC 단독 구현 가능성 | dept-objects 쿼리 작성 시 |
| `docs/references/keycloak-sync.md` | Keycloak 부서 그룹/사용자 동기화 절차 + 매칭 우선순위 + 트러블슈팅 | 동기화 스크립트 운영 시 / web-ui Keycloak 통합 사전 분석 시 |
| `references/external-connection-removal.md` | 외부 연결 제거(폐쇄망) 정책 + 코드 위치 매핑 (진행은 SESSION_HISTORY) | Marketplace/플러그인/외부연결 제거 작업 시 |

## 아키텍처 불변식

> 이 규칙을 위반하면 안 됩니다.

1. **AppMode 분기**: ADVANCED_CHAT은 `messages`만 읽음 — `workflow_runs` 이중카운트 금지 (H-DASH-01)
2. **디버깅 필터**: 모든 집계 쿼리에 `invoke_from != 'debugger'` + `triggered_from = 'app-run'` 필수 (H-DASH-03). audit 마트는 `details->>'invokeFrom' != 'debugger'` + `details->>'triggeredFrom' != 'debugging'` (collector 보강 후)
3. **RBAC 테이블**: 회사 표준 `spx_` 접두사 (2026-05-19 마이그레이션 `20260519000000_fix_layer1_mview_naming_and_rbac_prefix` 적용 — 코드 `__tablename__` 기준 SoT). `spx_departments` / `spx_department_members` / `spx_resource_ownership` / `spx_resource_permissions` / `spx_rbac_audit_logs`. ~~`sp_` 접두사(5/4 가정)~~ → ~~prefix 없음(5/6 가정)~~ → `spx_` (5/19 최종) 으로 2회 정정됨
4. **Dify 무수정**: 기존 Dify 테이블에 FK 추가 금지, 기존 코드 수정 최소화
5. **미배정 처리**: `spx_resource_ownership`에 없는 레거시 앱은 "미배정"으로 fallback (H-DASH-04)
6. **마트 입력**: audit_events 단일 SoT + RBAC JOIN (2026-05-12 이사님). 오브젝트 차트만 `spx_resource_ownership` 직접 조회 (state라 audit 본질 불가)

## 흔한 실수 → Harness 방어

| 실수 | ID | 방어 |
|------|-----|------|
| 토큰 이중카운트 | H-DASH-01 | ADVANCED_CHAT은 messages만 |
| 디버깅 데이터 혼입 | H-DASH-03 | invoke_from/triggered_from 필터 |
| 레거시 앱 누락 | H-DASH-04 | LEFT JOIN + "미배정" fallback |
| 이전 기간 데이터 삭제 | H-DASH-05 | 증감률 "N/A" 표시 |
| 증감률 NaN | H-DASH-07 | 분모 0 → "+신규" / "-" 분기 |
| NULL 사용자 카운트 | H-DASH-08 | from_end_user_id IS NOT NULL |

> 전체 패턴: `hdd/defect-catalog.md` 참조

## 응답 규칙

- **모든 응답은 한국어로 작성** (코드 식별자/주석은 영어 OK, 사용자에게 설명/안내는 한국어)
- LLM의 언어 누출(language leakage) 방지 — 긴 컨텍스트나 영문 명령어 다음 마무리 안내 문구에서 일본어/중국어로 빠지지 않게 의식적으로 한국어 유지

## 핵심 컨벤션

- Python: snake_case, Blueprint 등록, SQLAlchemy 모델 클래스
- TypeScript: camelCase, TanStack Query 훅, staleTime 5분
- 신규 API: `/console/api/dashboard/` 경로 하위에 등록 (2026-05-13 변경 — admin prefix 폐기)
- 신규 모델: `api/models/` 에 회사 표준 명명(`spx_` 접두사) 테이블 매핑
- TanStack queryKey: `['dashboard', '<component>', params]` — 페이지 헤더의 새로고침이 일괄 invalidate (2026-05-13 변경 — admin 세그먼트 폐기)
- 페이지 레벨 상태(기간 필터 등): URL query string 단일 진실 (Context/prop drilling 금지). 라우트 페이지라 query string 활용 가능
- 화면 레이아웃: 카드/차트/표 모두 고정 높이 + 콘텐츠 초과 시 영역 내부 스크롤. 데이터 양에 따라 카드 크기 변동되는 레이아웃 금지 (2026-05-13 신규 전역 정책)
- 디자인 토큰: raw 색상값 금지, `text-text-*` / `bg-components-*` / `util-colors-*` 토큰 사용
- 숫자 포맷: `<10K` raw + 콤마, `≥10K` K/M 압축 + hover 정확값 (전역 규칙, `hdd/design.md § 6.5` 참조)
