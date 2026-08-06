---
tags: [프로젝트, dify, AI-Agent, HDD]
type: harness/guide
date: 2026-04-29
last_updated: 2026-05-06
---
# Conventions — Dify 기존 코드 스타일에 맞추기

> 기존 Dify 패턴을 따라야 코드 리뷰 통과율이 높아진다.
> "이렇게 쓴다"만 적고, "왜"는 생략.

## 1. Python (백엔드)

### 네이밍

| 대상 | 규칙 | 예시 |
|------|------|------|
| 파일 | snake_case | `dashboard_kpi_service.py` |
| 클래스 | PascalCase | `DashboardKpiApi`, `DashboardKpiService` |
| 함수/변수 | snake_case | `get_kpi()`, `total_tokens` |
| 상수 | UPPER_SNAKE | `LOCAL_PROVIDERS`, `APP_MODE_MAP` |
| 테이블명 | snake_case + `spx_` 접두사 | `spx_departments`, `spx_resource_ownership` |
| 신규 테이블 | `spx_` 접두사 필수 (2026-05-19 확정 — ~~`sp_`~~ 폐기) | `spx_accounts`, `spx_department_members` |

### Controller 패턴 (Flask-RESTX Resource)

```python
from flask_restx import Resource  # Dify는 flask-restx 사용 (flask-restful 아님 — 헷갈리지 말 것)
from controllers.console import console_ns

@console_ns.route("/admin/dashboard/kpi")
class DashboardKpiApi(Resource):
    # method_decorators는 리스트 아래→위 순서로 실행
    method_decorators = [
        account_initialization_required,  # 3. 계정 초기화
        login_required,                   # 2. 로그인
        setup_required,                   # 1. 시스템 초기화
    ]

    @console_ns.doc("get_dashboard_kpi")
    @console_ns.response(200, "Success")
    def get(self):
        # HTTP 메서드명 = 함수명
        return DashboardKpiService.get_kpi(...)

    def post(self):
        pass
```

- 경로 파라미터: `<uuid:app_id>`, `<string:name>`
- HTTP 메서드: `get()`, `post()`, `put()`, `delete()`

### 데코레이터 체인

```python
@setup_required                    # 1. 시스템 초기화 확인
@login_required                    # 2. 로그인 확인 (JWT)
@account_initialization_required   # 3. 계정 초기화 확인
@edit_permission_required          # 4. 편집 권한 (선택)
@admin_required                    # 5. Admin 전용 (선택)
```

- 스택 실행: **아래→위** (가장 아래 데코레이터가 가장 먼저 실행)
- 어드민 API에는 `@admin_required` 추가 고려 (기존 `admin.py` 참고)

### Service 패턴

```python
class DashboardKpiService:
    @classmethod
    def get_kpi(cls, tenant_id: str, start: str, end: str) -> KpiResponse:
        # SQLAlchemy 직접 쿼리
        result = db.session.query(...).filter(...).all()
        return KpiResponse(total_objects=..., active_users=...)
```

- `@classmethod`로 정의 (인스턴스 불필요)
- DB 쿼리는 Service 안에서 직접 SQLAlchemy로 작성
- **반환 타입은 Pydantic 모델** (dict 직접 반환 금지) — 검증/직렬화 자동화
- 외부 HTTP 호출: `httpx` + `tenacity` 재시도

### Pydantic 응답 스키마 (Dify 컨벤션 따름)

```python
from pydantic import BaseModel

class KpiDetail(BaseModel):
    count: int
    diff_percent: float | None = None
    diff_label: str | None = None

class KpiResponse(BaseModel):
    total_objects: KpiDetail
    active_users: KpiDetail
    # ...

# Controller에서 직렬화
def get(self):
    response = DashboardKpiService.get_kpi(...)
    return response.model_dump(mode='json')  # datetime → ISO 자동
```

- 위치: `services/admin/<component>_schemas.py` (또는 `services/admin/schemas.py` 통합)
- 임포트: `from pydantic import BaseModel` (Dify 전반에서 사용 중 — `account_service.py`, `app_dsl_service.py` 등)
- TypeScript 인터페이스와 1:1 대응 (각 컴포넌트 design.md "Response 스키마" 섹션 참조)
- `mode='json'`: datetime → ISO 8601, UUID → str 자동 변환
- Optional 필드는 `field: T | None = None` (Python 3.10+ 문법)
- 상세 학습: [[4. 지식노트/Pydantic - Python 데이터 검증 라이브러리.md]]

### SQLAlchemy 쿼리 패턴

```python
# 집계
db.session.query(
    func.count(Message.id).label("count"),
    func.sum(Message.message_tokens + Message.answer_tokens).label("tokens"),
).filter(
    Message.app_id == app_id,
    Message.invoke_from != InvokeFrom.DEBUGGER.value,  # H-DASH-03
).first()

# CTE (부서별 활동 등 복잡 쿼리)
cte = db.session.query(...).cte("dept_apps")
result = db.session.query(cte.c.dept_name, ...).all()
```

### Fallback 패턴 (`spx_` 테이블 미생성 등)

`try/except` fallback 사용 시 **except 첫 줄에 `db.session.rollback()` 필수**. PostgreSQL은 트랜잭션 안에서 한 번 SQL 에러가 나면 rollback 전까지 후속 쿼리를 모두 거부함 (`InFailedSqlTransaction`).

```python
def _get_dept_counts(...):
    try:
        return db.session.query(Department).filter(...).all()
    except Exception:
        db.session.rollback()    # ← 안 하면 다음 쿼리/fallback이 폭발
        logger.exception("spx_departments fallback")
        return []
```

- 단일 쿼리 service에선 잠복 (rollback 안 해도 한 번만 실패하니까)
- **여러 쿼리 순차 호출** 또는 **except 안 fallback SQL** 사용 시 발현
- 단위 테스트 mock 환경에선 트랜잭션 시뮬레이션 안 돼서 잠복 → **통합 테스트로만 잡힘**
- 관련 결함: H-DASH-04 (`spx_` 테이블 미생성 fallback)
- 상세: [[4. 지식노트/PostgreSQL - InFailedSqlTransaction과 SQLAlchemy fallback 패턴]]

#### `logger.exception` 의무 — silent fallback 함정 방어

`except Exception: return []` 패턴에 `logger.exception` 누락하면 **에러가 발생해도 로그에 안 찍혀 디버깅 불가**. 5/7 사례: service 4종 명명 정정 후 화면이 빈 데이터로 보였는데 silent fallback 때문에 진짜 원인 추적에 1시간+ 소요. (SQL은 정상이었고 gunicorn worker 재시작 필요한 상황이었음 — `logger.exception`만 박혀있었으면 `docker logs api | grep ERROR` 한 줄로 즉시 확인 가능).

```python
# ❌ Silent — 디버깅 함정
except Exception:
    db.session.rollback()
    return []

# ✅ 표준 — 위 Fallback 예시처럼 logger.exception 의무
except Exception:
    db.session.rollback()
    logger.exception("Failed to fetch <메서드명> — falling back to empty")
    return []
```

**기존 코드 적용 검증** (신규 컴포넌트 추가 시 또는 정기 점검):

```bash
# silent fallback이 남아있는 파일 찾기
grep -rln 'except Exception' api/services/ | xargs grep -L 'logger.exception'
```

결과 0건이면 OK. 출력된 파일은 보강 필요.

### Mock 패턴

```python
from unittest.mock import MagicMock, patch

# Redis
redis_mock = MagicMock()
patch.object(ext_redis, "redis_client", redis_mock)

# 환경변수
monkeypatch.setenv("DB_TYPE", "postgresql")
```

### 로깅 패턴 (ruff TRY400 강제)

except 블록 안에선 **`logger.exception()` 사용**. `logger.error()` 금지.

```python
import logging
logger = logging.getLogger(__name__)

# ❌ Bad — 메시지만 남고 traceback이 사라짐
try:
    do_something()
except Exception as e:
    logger.error(f"failed: {e}")

# ✅ Good — traceback 자동 첨부
try:
    do_something()
except Exception:
    logger.exception("failed")
```

- `.exception()` = `.error()` + `exc_info=True` (스택 트레이스 자동)
- ruff `TRY400` 룰이 자동 검출 + `--fix`로 자동 치환 가능
- except 블록 *밖*에서는 `.error()` 정상 사용 (스택 없음)
- 상세: [[4. 지식노트/Python - logger.error vs logger.exception (ruff TRY400)]]

### datetime UTC (ruff UP017, Python 3.11+)

`from datetime import UTC` 사용. `timezone.utc`는 구식.

```python
# ❌ Bad (3.10 이하 호환, 3.11+에서도 작동은 함)
from datetime import datetime, timezone
now = datetime.now(timezone.utc)

# ✅ Good (Dify는 Python 3.11+ 요구)
from datetime import datetime, UTC
now = datetime.now(UTC)
```

- `datetime.UTC`는 `timezone.utc`와 **완전 동일 객체** (`UTC is timezone.utc` → `True`)
- 더 간결, 신코드 표준
- ruff `UP017` 룰이 자동 fix
- 상세: [[4. 지식노트/Python - datetime.UTC alias (Python 3.11+, ruff UP017)]]

### Lint 룰 — 강제되는 것 요약

| 룰 ID | 의미 | 자동 fix |
|-------|------|---------|
| TRY400 | except 블록 `logger.error` → `logger.exception` | ✅ |
| UP017 | `timezone.utc` → `UTC` | ✅ |
| COM812 | 함수 인자/dict 마지막 trailing comma 강제 | ✅ |
| E501 | 줄 길이 초과 (120자) | ❌ 수동 |

`pnpm install` 후 `.vite-hooks/_` 활성화되면 commit 시 자동 검사. **신규 환경에서 hook 미활성 시 사후 발견 가능** — `architecture.md § 9` 참조.

---

## 2. TypeScript (프론트엔드)

### 네이밍

| 대상 | 규칙 | 예시 |
|------|------|------|
| 파일 (컴포넌트) | kebab-case | `kpi-card.tsx`, `dept-activity-table.tsx` |
| 파일 (훅) | kebab-case, `use-` 접두 | `use-admin-dashboard.ts` |
| 컴포넌트 | PascalCase | `KpiCard`, `DeptActivityTable` |
| 훅 | camelCase, `use` 접두 | `useAdminDashboardKpi()` |
| 인터페이스 | PascalCase | `KpiResponse`, `DeptObjectsItem` |
| 상수 | UPPER_SNAKE | `ACCOUNT_SETTING_TAB` |
| CSS | Tailwind utility classes | `className="grid grid-cols-2 gap-4"` |

### 컴포넌트 구조

```
web/app/components/admin/kpi-section/
├── index.tsx           ← 메인 컴포넌트 (export default)
├── kpi-card.tsx        ← 하위 컴포넌트
└── types.ts            ← 타입 정의 (선택)
```

- `index.tsx`를 메인 진입점으로 사용
- 테스트: 같은 폴더에 `__tests__/index.spec.tsx`

### TanStack Query 훅 (2026-05-13 admin 세그먼트 폐기)

```typescript
// web/service/use-admin-dashboard.ts
import { useQuery } from '@tanstack/react-query'
import { get } from './base'

export const useAdminDashboardKpi = (params?: DateRangeParams) => {
  return useQuery<KpiResponse>({
    queryKey: ['dashboard', 'kpi', params],
    queryFn: () => get<KpiResponse>('/dashboard/kpi', { params }),
    staleTime: 5 * 60 * 1000,  // 5분 캐시
  })
}
```

- `queryKey` 배열: `['dashboard', '<component>', params]` 패턴
- 페이지 헤더 새로고침 = `queryClient.invalidateQueries({ queryKey: ['dashboard'] })`로 일괄 무효화
- `staleTime: 5분` — 대시보드 데이터 기본 캐시
- 기존 패턴 참고: `web/service/use-apps.ts`

### ECharts 차트

```typescript
import ReactECharts from 'echarts-for-react'

// 옵션 빌더 유틸 활용 (기존 패턴)
const option = buildChartOptions({
  xAxis: { data: labels },
  series: [{ type: 'bar', data: values }],
})

return <ReactECharts option={option} style={{ height: 300 }} />
```

- 라이브러리: `echarts-for-react`
- 팩토리: `createBizChartComponent()` 또는 직접 `<ReactECharts />`
- **색상 (전역 매핑 — 모든 컴포넌트에서 통일)**:
  - **App** = `util-colors-blue-blue-500` (파랑)
  - **KB** = `util-colors-teal-teal-500` (청록)
  - **Tool** = `util-colors-orange-orange-500` (주황)
  - 로컬 모델 = `util-colors-teal-teal-500` (다른 모델은 blue)
  - 시맨틱: `text-text-success` (상승) / `text-text-destructive` (하락) / `text-text-tertiary` (변동 없음)
- raw 색상값(`#xxxxxx`, `rgb()`) 금지 — 디자인 토큰만 사용 (다크모드 자동 대응)
- 상세: [[4. 지식노트/Dify - 디자인 토큰과 차트·카드 색상 패턴.md]]
- 레이아웃: `grid-cols-1 xl:grid-cols-2` (2컬럼 그리드)
- 카드 컨테이너 표준 (2026-05-14 갱신 — 2분리): **메인 카드 (KPI)** = 스튜디오 `AppCard` 패턴 / **컨테이너 카드 (차트·표)** = 스튜디오 `NewAppCard` 패턴. 정확한 className은 `web/app/(commonLayout)/apps/` 코드에서 추출. 이전 단일 패턴(모달 PROVIDER 톤)은 라우트 페이지 이동(5/13)으로 폐기. 상세 근거: [[3. 프로젝트/spx-agent/hdd/design.md|design.md § 6.5]]

### 상태 관리

| 목적 | 도구 | 위치 |
|------|------|------|
| API 캐시 | TanStack Query | `web/service/` |
| 앱 상세 상태 | Zustand | `web/app/components/app/store.ts` |
| 인증/워크스페이스 | Context | `web/context/app-context-provider.tsx` |
| 모달 상태 | nuqs (URL 파라미터) | `web/context/modal-context-provider.tsx` |

### 헤더 톱 네비 메뉴 추가 (2026-05-13 신설 — 대시보드 라우트 이동)

대시보드는 `/dashboard` 톱레벨 라우트로, 헤더 톱 네비 첫 번째 메뉴(`DashboardNav`)로 추가됨. ExploreNav 패턴 미러.

**① `web/app/components/header/dashboard-nav/index.tsx` — 신규**
```typescript
import { RiDashboardFill, RiDashboardLine } from '@remixicon/react'
import { useTranslation } from 'react-i18next'
import { useSelectedLayoutSegment } from 'next/navigation'
import NavLink from '../nav-link'

const DashboardNav = () => {
  const { t } = useTranslation()
  const segment = useSelectedLayoutSegment()
  const isActive = segment === 'dashboard'

  return (
    <NavLink
      href="/dashboard"
      icon={isActive ? <RiDashboardFill /> : <RiDashboardLine />}
      label={t('common.menus.dashboard')}
      isActive={isActive}
    />
  )
}
```

**② `web/app/components/header/index.tsx` — 첫 번째 위치 추가**
```typescript
<DashboardNav />     // ← 추가 (첫 번째)
<ExploreNav />
<AppNav />
<DatasetNav />
<ToolsNav />
```
5개 메뉴 균등 가운데 배치 (구현 시 정렬 확인 필요).

**③ i18n 키 추가** — `web/i18n/ko-KR/common.ts` + `en-US/common.ts`
```typescript
menus: {
  dashboard: '대시보드',  // ko-KR
  // dashboard: 'Dashboard',  // en-US
}
```

**④ 로그인 후 디폴트 redirect 변경** — `web/app/signin/utils/post-login-redirect.ts`
```typescript
export const DEFAULT_POST_LOGIN_PATH = '/dashboard'
```
7군데 하드코딩된 `'/apps'` fallback을 일괄 교체.

**⑤ 설정 모달 DASHBOARD 탭 제거** — `web/app/components/header/account-setting/`
- `constants.ts`에서 `ACCOUNT_SETTING_TAB.DASHBOARD` 제거
- `index.tsx`에서 사이드바 메뉴 항목 + 렌더 분기 제거
- `dashboard-page/` 폴더 삭제

### 마운트 환경 정책 (2026-05-14 갱신: 페이지 폭 + h1 제거)

대시보드 페이지 작성 시:
- **자체 `<h1>` 페이지 헤더 미작성** (2026-05-14 변경 — 첫 실측 후 톱 네비에 이미 "대시보드"가 활성 상태로 표시되어 페이지 내 h1은 중복. 스튜디오/지식 페이지도 자체 h1 없음)
- **페이지 폭 = commonLayout 표준 컨테이너 따름** (2026-05-14 신설 — 스튜디오/지식과 동일한 좌우 여백/최대 폭). 우리 페이지가 전체 폭을 채우는 자체 래퍼 만들지 말 것. `web/app/(commonLayout)/apps/` 또는 `datasets/` 페이지의 외곽 컨테이너 패턴을 그대로 차용
- **URL query string 활용** — 기간 필터, drill-through deep link 등은 URL 단일 진실. `nuqs` 패키지 활용 가능
- **ESC 키 자유 활용** — drill-through 해제 등 (모달 자동 닫기 X)
- **`dashboard-controls`는 페이지 우상단에 단독 배치** (h1 제거로 헤더 슬롯이 사라짐 — 페이지 상단 우측 정렬, 좌측은 비움). 5/13 "페이지 헤더 슬롯" 표현 폐기
- **비로그인 가드 자동 적용**: `(commonLayout)` 하위의 `AppInitializer`가 → `/signin`

### 화면 레이아웃 정책 (2026-05-13 신규 전역 정책)

- 카드/차트/표 모두 **고정 높이**
- 콘텐츠 초과 시 **영역 내부 스크롤** (외부로 빠지지 않음)
- 데이터 양에 따라 카드 크기 변동되는 레이아웃 **금지**
- 표는 헤더 고정 + 본문 스크롤
- **본문 고정 높이 (2026-06-09 구현)**: 카드 본문에 `max-h-*` 대신 **고정 `h-*`** 부여 → 빈↔데이터 전환 시 카드 크기 불변. 빈/에러 상태도 **데이터와 같은 높이**(작은 `h-40`/`h-[160px]` 금지). 표준값:
  - 드릴 차트(좌하/우하 막대): 본문 `h-[250px]`
  - 표(드릴 + 부서별 활동): 본문 `h-[392px]` (빈 메시지는 `align-middle`로 세로 중앙)
  - 모델별 토큰 차트: `h-[260px]` / 부서별 오브젝트 차트: `h-[280px]`

### 빈 상태 / 골격 / 결측치 정책 (2026-06-09 신규 — SoT)

> 데이터가 없거나 일부 결측일 때 표/차트가 어떻게 보여야 하는지의 **단일 진실**. 다른 문서(design/quality-criteria/specs)는 이 절을 참조만 한다.

**원칙 1 — 골격 항상 유지 (제목 날리기 금지)**
- 데이터가 없어도 **제목**(표는 +컬럼 헤더, 디멘전 차트는 +축 이름)은 **항상** 렌더한다.
- ❌ 금지: `if (items.length === 0) return <맨 "데이터 없음" 박스>` 처럼 **위젯 통째(제목 포함)를 early-return으로 갈아치우기**. → 제목은 빈 상태 분기 **밖**에 두고, 빈 처리는 **몸통만** 한다.
- 빈 몸통 안내 문구는 가능하면 **기간 명시**: 예) `{기간} 동안 호출된 앱이 없습니다`. (막연한 "데이터가 없습니다" 지양)

**원칙 2 — 결측치 = `-`**
- 셀/값이 `0` 또는 없음 → 회색 `-` (`text-text-tertiary`). (§ 추세 기호 표와 일치)
- 계산 불가(이전 기간 데이터 삭제 등) → `N/A` (KPI 증감률에 한정).

**원칙 3 — 전수(show-all) vs 활성(used-only) 판단 프레임워크**

표/차트가 "전체를 깔지, 사용된 것만 보일지"는 아래 5질문으로 결정한다:

| 질문 | → 전수 | → 활성만 |
|------|--------|----------|
| 개수가 작고 안정적인가? | 예(수십 이하 고정) | 아니오(무한 증가) |
| 고정 분류축 vs 엔티티? | 분류축 | 엔티티 |
| "0/없음"이 그 자체로 정보인가? | 예(미도입 신호) | 아니오(노이즈) |
| 전체 목록 보는 다른 경로 有? | 무 | 유 |
| 제목/진입점 함의 | "X별"(전수) | "인기/Top/랭킹" |

- **부서**(고정 분류축) → **전수**: 활동 0 부서도 행/축에 유지(값 `-`, 차트는 0막대). 백엔드가 `spx_departments` 활성 전체를 시드(쿼리 결과에 overlay).
- **앱·사용자·모델**(엔티티/랭킹) → **활성만** + 기간 명시 안내 문구. (모델은 "설정된 전체 모델" 명부가 마트/RBAC에 없어 문구로 처리)
- **"미배정" sentinel 행은 항상 마지막** (§ 정렬 규칙 예외와 일치).

> ⚠️ **검증 후 확정 (TODO, 2026-06-09)**: 디멘전 *차트*의 "이름축 + 0막대" 시각은 아직 화면 검증 전. 0막대가 어색하면 안내 문구로 전환 가능 — 프로토타입 1개 확인 후 본 항목 확정.

> **적용 현황(2026-06-09)**: 표 5종(dept-activity/dept-new-creations/dept-users/dept-call-rps/app-stats) + 차트 10종(빈 상태 골격/문구) 완료. 부서 디멘전 차트 0막대 시각은 **화면 검증 후 위 TODO 확정 예정**. 다음 = 고정 높이(빈↔데이터 크기 변동 제거, § 화면 레이아웃 정책 구현). 검증된 레퍼런스 예시 = [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md]].

### 인터랙션 정책 (2026-05-13)

- **좌하 차트 클릭**: 동작 없음 (차트 드로어 보류로 트리거 제거). 차트만 표시
- **표 행 클릭**: KPI 4번(앱별 통계)만 → 같은 탭으로 Dify 모니터링 페이지(`/app/{appId}/overview`). KPI 1·2·3 표 클릭은 없음
- **KPI 카드 클릭**: drill-through 펼침 (Phase 2)

---

## 3. 테스트

### 백엔드 (pytest)

**폴더 구조:**
```
api/tests/
├── unit_tests/           ← SQLite in-memory, Redis mock
│   ├── conftest.py       ← Flask 앱, 자동 리셋 fixture
│   └── services/admin/   ← 소스 구조 반영
└── integration_tests/    ← 실제 DB, JWT 인증
    ├── conftest.py       ← test_client, auth_header fixture
    └── controllers/console/admin/
```

**네이밍:** 파일 `test_*.py`, 함수 `def test_*()`, 클래스 `class Test*:`

**fixture 패턴:**
```python
@pytest.fixture(autouse=True)
def _reset_redis(redis_mock):
    redis_mock.reset_mock()

@pytest.fixture
def auth_header():
    # JWT 토큰 포함된 인증 헤더
    return {"Authorization": f"Bearer {token}"}
```

**실행:**
```bash
pytest api/tests/unit_tests/
pytest api/tests/integration_tests/
```

### 프론트엔드 (Vitest)

**위치:** `web/app/components/기능명/__tests__/index.spec.tsx`

**패턴:**
```typescript
import { vi } from 'vitest'

vi.mock('@/context/provider-context', () => ({
  useProviderContext: vi.fn(),
}))

describe('KpiCard', () => {
  beforeEach(() => vi.clearAllMocks())
  it('should render kpi value', () => { ... })
})
```

**실행:**
```bash
cd web
pnpm test              # 실행
pnpm test:coverage     # 커버리지
pnpm test:watch        # 감시 모드
```

---

## 4. 공통 규칙

| 규칙 | 설명 |
|------|------|
| **Dify 무수정** | 기존 Dify 파일 수정 최소화. 신규 모듈만 추가 |
| **기존 토큰/컴포넌트 우선** | 신규 className/컴포넌트 만들기 전에 기존 패턴 먼저 확인. (a) 디자인 토큰 — `web/themes/`, 다른 탭의 className grep / (b) 베이스 컴포넌트 — `web/app/components/base/` / (c) 동일 패턴이 모달 내 다른 탭에 있으면 그걸 따름 (모달 톤 통일). 새 토큰/컴포넌트 도입은 **기존에 없을 때만**, 도입 시 design.md 또는 conventions.md에 표준으로 등록 |
| **`spx_` 접두사** | 신규 DB 테이블은 반드시 `spx_` 접두사 (2026-05-19 확정 — ~~`sp_`~~ 폐기) |
| **디버깅 필터** | 모든 집계 쿼리: `invoke_from != 'debugger'` + `triggered_from = 'app-run'` |
| **staleTime 5분** | 대시보드 API 훅의 기본 캐시 |
| **queryKey 표준** | `['admin', 'dashboard', '<component>', params]` — 헤더 새로고침 일괄 invalidate |
| **숫자 포맷** | `<10K` raw + 콤마, `≥10K` K/M 압축 + hover 정확값 (전역 규칙) |
| **에러 상태** | 로딩/에러/데이터없음 3상태 항상 처리. **빈 상태도 제목/축 골격 유지 + 결측치 `-`** → § 빈 상태 / 골격 / 결측치 정책 참조 (early-return으로 제목까지 날리기 금지) |
| **마운트 환경** | 대시보드는 설정 모달 탭 — 자체 `<h1>` 금지, React state 사용 |
| **CSV 내보내기** | 정적 대시보드엔 없음. Phase 2 우측 드로어에서 화면만 (백엔드 미구현) |

---

## 5. 작성 워크플로우

### 패턴 복제 전 1개 lint (AI 작성자 함정 방어)

같은 패턴을 여러 파일에 복제할 때(예: 차트 N종, 표 N종, service 메서드 N개)는 **첫 1개 작성 직후 lint 1회 통과**시킨 뒤 나머지 복제할 것.

```bash
pnpm eslint <파일경로> --fix     # 프론트
ruff check <파일경로> --fix      # 백엔드
```

**기준 우선순위**: 패턴 복제 전 1회 > 디렉토리 단위 > 파일 단위 > commit 직전.

**근거**: AI 작성자(VSCode Claude/Codex 등)는 lint 자체검증을 안 함. 잘못된 패턴이 첫 파일에 박히면 복제로 N배 증식. commit 시점 hook은 정상 작동하지만 그 시점엔 N개 일괄 수정 부담 (5/7 사례: drill-charts 8종에 동일한 `any` + `2 statements per line` 박힘 → commit 직전 hook 발동으로 일괄 발견).

### 신규 endpoint/service 추가 후 컨테이너 재시작 의무

백엔드에 신규 service 파일이나 controller endpoint를 추가한 뒤, Docker 환경에서 확인하려면 **반드시 컨테이너 재시작** 필요. gunicorn은 코드 변경을 자동 reload하지 않으므로 신규 모듈을 import하지 못해 404가 됨.

```bash
docker compose restart api worker worker_beat
```

**증상**: 코드는 정상인데 endpoint가 404 → gunicorn reload 누락이 원인 (5/7 drill-through 11종 추가 시 발견).

### 자주 박히는 패턴 — ECharts 콜백

차트 컴포넌트의 `tooltip.formatter` / `label.formatter` 콜백에서 params 타입을 `any`로 박으면 `ts/no-explicit-any` 위반. **정식 타입 사용**:

```typescript
import type { TooltipComponentFormatterCallbackParams } from 'echarts'

formatter: (params: TooltipComponentFormatterCallbackParams) => `${params.name}: ${params.value}`
```

inline `as any` 캐스팅 금지 — import해서 명시적 타입으로.

### 자주 박히는 패턴 — Tooltip (deprecated 마이그레이션)

`@/app/components/base/tooltip`은 **deprecated** (dify upstream issue #32767). 새 import 사용:

```typescript
// 옛 (deprecated, 사용 금지)
import Tooltip from '@/app/components/base/tooltip'
<Tooltip popupContent={X}><Y /></Tooltip>

// 새 (named exports, Radix-style 분리)
import { Tooltip, TooltipContent, TooltipTrigger } from '@/app/components/base/ui/tooltip'
<Tooltip>
  <TooltipTrigger><Y /></TooltipTrigger>
  <TooltipContent>{X}</TooltipContent>
</Tooltip>
```

- `TooltipProvider`는 `app/layout.tsx` 최상단에 이미 박혀있어 추가 wrap 불필요.
- 단순 트리거(아이콘/span)는 `<TooltipTrigger>` children 패턴으로 충분. 외부 컴포넌트(예: `Avatar`)에 props 패스가 필요할 때만 `render={(props) => ...}` 사용 (예: `members-page/index.tsx` line 62~81).
- dify 본체에 남은 옛 import는 우리가 손대지 않음 — Dify 무수정 원칙. upstream에서 마이그레이션 진행 중.

### 컨테이너 환경 디버깅 — "호스트 ↔ 컨테이너 코드 drift" 첫 단계 검증

화면/endpoint 동작 안 함 증상 시 **첫 5분에 의무**:

```bash
# 1) 호스트 코드와 컨테이너 안 코드 일치 여부
docker exec <container> grep -c "<우리 새 코드 시그니처>" /app/<path>
# 0이면 코드 미반영 → 재시작 또는 rebuild 결정

# 2) 컨테이너 안 디렉토리 구조 자체 확인 (production build인지 source인지)
docker exec <container> ls /app/<expected-source-dir>
# "No such file or directory"면 build된 이미지라 source 통째 없음
```

**서비스별 변경 후 액션 매트릭스**:

| 변경 영역 | 컨테이너 반영 메커니즘 | 검증 |
|---|---|---|
| `api/services/`, `api/controllers/` | `docker-compose.override.yaml` 통째 mount | gunicorn worker reload 안 됨 → **`docker compose restart api worker worker_beat`** |
| `web/app/`, `web/service/` | **mount 없음** (production build 이미지) | **`docker compose build web` (5~10분) → up -d web** |
| `api/migrations/versions/` | mount 통해 즉시 반영 | `docker exec <api> flask db heads`로 검증 |

**5/7 사례 — 같은 가족 함정 하루 3건**:
- 1차 (오전): service 명명 정정 → 화면 빈 데이터 → gunicorn worker가 옛 모듈 캐시. 진짜 원인은 silent fallback (logger.exception 누락) + worker reload. stop+start로 해결.
- 2차 (오후 초반): 백엔드 11종 endpoint 404 → 같은 worker reload 누락. stop+start로 해결.
- 3차 (오후 후반): 프론트 연동 후 mock 그대로 → web 컨테이너에 호스트 코드 반영 자체 안 됨 (production build 이미지). image rebuild로 해결.

**메타 — 디버깅 첫 단계 의무**: 1, 2번을 stop+start로 해결한 게 진짜 본질(컨테이너 안 코드 검증)을 못 잡게 함. 3번에서야 *"코드가 컨테이너에 있긴 한가?"* 질문이 등장. 다음에는 1번 시점부터 위 첫 단계 검증 박을 것.

### 프론트 변경 후 build 의무 (회사 워크플로우)

`web/` 디렉토리 변경 후 화면 검증 시:

```bash
docker compose -f docker/docker-compose.yaml build web
docker compose -f docker/docker-compose.yaml up -d web
```

**근거**: web 서비스는 `Dockerfile`로 production build 이미지 사용 (`docker-compose.yaml`의 `build:` 지시). 호스트 코드 변경 → mount 없음 → 컨테이너 안엔 build 시점의 옛 코드 그대로. 5~10분 소요지만 검증 신뢰 위해 필수.

api/worker는 mount 통해 자동 반영(restart만 필요), web은 build 필요 — 결을 분리해서 외울 것.

### 보고 — 프롬프트의 모든 작업 항목 처리 여부 명시

여러 단계 작업을 수행한 뒤 보고할 때는 **프롬프트에 박힌 모든 작업 항목 각각에 대해 "처리 / 미처리 / 다음 단계"를 명시**할 것. 안 한 항목을 침묵으로 넘기면 사용자가 화면 검증할 때야 발견됨.

```markdown
[보고 형식 예]
1. 백엔드 service 11종 — ✅ 완료 (44 테스트 통과)
2. silent fallback 보강 — ✅ 완료 (4파일 logger.exception 추가)
3. 프론트 fetch 훅 — ⏳ 미처리 (다음 단계로 분리)
4. drill-charts dummy mock 제거 — ⏳ 미처리 (3과 함께)
```

**근거**: 5/7 백엔드 11종 작업 시 프롬프트에 [프론트 연동] 항목 포함됐으나 보고에서 *"web/ 영역 작업 0건"* 사실 미언급. 사용자가 화면 검증 후 *"drill-through가 안 보임"* 단계에서야 프론트 연동 누락 발견. 보고 시점에 한 줄 명시했으면 즉시 다음 프롬프트 던지기 가능.

**원칙**: 침묵 = 미처리 추정 X. **명시 = 안전**. 작업 범위가 큰 프롬프트일수록 보고 체크리스트 의무.
