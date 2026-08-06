---
tags: [프로젝트, dify, AI-Agent, HDD]
status: 초안
date: 2026-04-28
last_updated: 2026-05-15
upstream:
  - "[[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]"
  - "[[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]"
  - "[[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]]"
  - "[[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]]"
downstream:
  - "[[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]]"
change_rules:
  - RBAC 스키마 변경 → 집계 쿼리 전체 재검토
  - AppMode 토큰 저장 위치 변경 → KPI 합산 수정
  - Keycloak upsert 매핑 키 변경 → 사용자↔부서 쿼리 재검토
  - 설정 모달 사이드바 합의 변경 → 충돌 대응 재확인
---
# SPX-Agent 대시보드 - HDD 상세 설계

> 이 문서는 **변경 영향 분석의 기준점**이다.
> "이게 바뀌면 저기도 바뀐다"를 선언해서, 변경이 생겼을 때 어디를 같이 고쳐야 하는지 즉시 추적할 수 있게 한다.

## 1. 시스템 경계

### 이번 범위 (In Scope) — 2026-05-13 갱신

| 구분 | 내용 |
|------|------|
| 대시보드 페이지 | `/dashboard` 톱레벨 라우트. KPI 카드 4종(총 오브젝트 / 부서별 채택 앱 수 / API 호출 / 앱별 통계), 부서별 오브젝트 차트, 모델별 토큰 차트, 부서별 활동 테이블 |
| 대시보드 컨트롤 | 페이지 헤더 슬롯에 배치할 기간 선택 + 새로고침 컨트롤. 자체 `<h1>` 페이지 헤더 작성 |
| 헤더 톱 네비 | `DashboardNav` 첫 번째 메뉴 추가 (5개 메뉴 균등 가운데 배치). 아이콘 `RiDashboardFill`/`RiDashboardLine` |
| 로그인 후 디폴트 | `/dashboard` 단독 (`DEFAULT_POST_LOGIN_PATH` 상수, 7군데 일괄 교체) |
| 설정 모달 수정 | 기존 DASHBOARD 탭 제거 + `dashboard-page/` 폴더 삭제 |
| 백엔드 API | `/console/api/dashboard/` 하위 집계 API + drill-through 11종 |
| 마트 입력 | `spx_audit_events` 단일 SoT + RBAC JOIN. 오브젝트 차트만 `resource_ownership` 직접 |
| 캐시 | TanStack Query staleTime (5분) |
| 상태 관리 | URL query string + React state (라우트 페이지 환경) |

### 이번 범위 밖 (Out of Scope) — 2026-05-13 갱신

| 구분 | 이유 |
|------|------|
| 중앙 통제 액션 (비활성/이관/삭제, 토큰 한도, 알림) | 나중에 별도 추가 |
| RBAC 테이블 생성/마이그레이션 | ✅ 완료 (권대리님 `feat/rbac` 브랜치, 2026-05-06) |
| SSO 인증 흐름 / Keycloak upsert | 승랑님 담당 |
| 감사로그 화면 | 승랑님 담당 (`dify-audit` 별도 앱) |
| 차트 드로어 4종 | 🔒 보류 (2026-05-13 이사님). 좌하 차트 클릭 동작 없음, 차트만 표시. 재검토 시점 미정 |
| CSV 내보내기 | 차트 드로어 보류와 함께 현재 미구현 |

## 2. 의존 관계 맵 (2026-05-13 갱신)

```
[외부 의존]

  spx_audit_events (단일 SoT) ──┐
  + RBAC 4종 (회사 표준 명명)    │
    departments                   │
    department_members            ▼
    resource_ownership ──→ 대시보드 백엔드 API ──→ 대시보드 프론트엔드 (/dashboard)
    resource_permissions    ▲
                            │
  audit collector 14종 ─────┘
  (messages, workflow-runs,
   workflow-nodes, conversations,
   nginx log-watcher, pg_trigger, ...)

  Dify OLTP (messages, workflow_runs) — 거의 직접 참조 안 함 (마트=audit 단일 SoT)
  단 오브젝트 차트는 resource_ownership 직접 (state 본질)
```

### 의존별 영향과 대응

| 의존 대상 | 변경 시 영향 | 대응 |
|-----------|------------|------|
| **spx_audit_events 스키마 / collector** | 마트 ETL 입력 전체 → 모든 KPI·차트 쿼리 영향. P0 보강 ✅ 완료 (2026-05-15). collector 14종 INSERT target rename + collector 4건 patch + Generated Column 8개 박혔음 | `references/audit-details-spec.md` 매트릭스. H-CAND-audit-appmode-missing / wf-debug-filter-missing 해소 |
| **RBAC 테이블 스키마** | 집계 쿼리 + 오브젝트 차트 → H-DASH-13 | SQLAlchemy 모델 클래스로 격리. 회사 표준 명명 박힘 (2026-05-06) |
| Dify `messages` 구조 | 거의 영향 없음 (마트=audit) | audit collector가 messages 변경 흡수 |
| Dify `workflow_runs` 구조 | 거의 영향 없음 (마트=audit) | 동일 |
| **resource_ownership** | 오브젝트 차트 3종 직접 의존 | `references/objects-charts-feasibility.md` 검증 완료. 운영 시 apps LEFT JOIN 패턴 (H-DASH-04) |
| ~~Keycloak upsert~~ | ~~H-DASH-14~~ | ✅ 자동 해소 (2026-05-06 — accounts.id 직접 매핑) |
| ~~설정 모달 사이드바~~ | ~~H-DASH-15~~ | ✅ 종결 (2026-05-13 라우트 이동으로 모달에서 빠짐) |

## 2.5. 마트 설계 (spx_audit_events 단일 SoT, 2026-05-15 확정)

> **2026-05-15**: rename 적용 완료 (`audit.audit_events` → `public.spx_audit_events`, audit schema 폐기) + collector P0 4건 패치 적용 + Generated Column 8개(`_d` 접미사) + 인덱스 5개 마이그레이션 완료. 현재 위치 = **Layer 1 진입 직전**. 정식 명세는 [[3. 프로젝트/spx-agent/references/audit-details-spec.md]] 참조.

> 5개 컴포넌트(kpi-cards / kpi-drill-through / dept-objects / dept-activity / model-tokens) 한정. spx_audit_events에 Generated Column 8개를 박고 → enriched MView 1장(audit × RBAC ID-only 사전 JOIN, 플래그 컬럼화) → 그 위에 daily roll-up MView 2장. 부서명은 항상 query-time JOIN으로 즉시 반영, 비싼 ownership/dept_members 결합은 refresh 시 1회 흡수. 채택은 actor, 호출은 owner.

### 2.5.0 명명 규칙 — `public` schema + `spx_` prefix (2026-05-14 결정, 2026-05-15 rename 적용 완료)

우리가 만드는 audit 테이블 + 모든 마트 객체는 **모두 `public` schema에 두고 `spx_` prefix로만 구분**. 별도 schema(`audit`/`mart`) 폐기. 회사 RBAC 표준 5종(`spx_departments`, `spx_department_members` 등 — 2026-05-19 `spx_` 접두사로 통일)과 우리 마트 객체가 모두 `spx_` prefix를 공유하고, Dify upstream 객체(`apps`, `messages`, `workflow_runs` 등 — 무접두)만 prefix로 구분되어 통일.

| 영역 | 적용 |
|---|---|
| audit 테이블 | ~~`audit.audit_events`~~ → **`public.spx_audit_events`** ✅ rename 완료 (5/15) |
| audit 부수 테이블 | `audit.collector_state/log_file_state/system_logs/system_log_batch` → **`public.spx_collector_state` 등** ✅ rename 완료 (5/15) |
| audit trigger 함수 | ~~`audit.log_dify_change()`~~ → **`public.spx_log_dify_change()`** ✅ rename 완료 (5/15) |
| Layer 1 마트 | ~~`mart.v_audit_enriched`~~ → **`public.spx_mv_audit_enriched`** / ~~`mart.v_resource_ownership_enriched`~~ → **`public.spx_v_resource_ownership_enriched`** (생성 예정) |
| Layer 2 마트 | ~~`mart.mv_kpi_calls_daily`~~ → **`public.spx_mv_kpi_calls_daily`** / ~~`mart.mv_model_tokens_daily`~~ → **`public.spx_mv_model_tokens_daily`** (생성 예정) |
| 마트 refresh 트리거 | **collector polling cycle 끝에 chain refresh 호출** (2026-05-15 결정 변경 — pg_cron 미채택). job 이름 X. § 2.5.5 참조 |

> 본 § 2.5 본문의 DDL/다이어그램/표는 모두 새 명명 기준으로 작성됨. 객체 이름에 `public.` prefix는 PostgreSQL search_path 기본값이라 통상 생략 가능 (DDL 작성 시 명시는 선택). audit schema는 `DROP SCHEMA ... CASCADE`로 폐기 (5/15 사고 학습: 비-CASCADE는 silent fail 위험 — `setup_audit_schema.sql`에 CASCADE 박음).

### 2.5.1 기본 분기 3건 + RBAC JOIN 옵션 B

| # | 항목 | 결정 |
|---|---|---|
| a | "신규 X" 정의 | `resource_ownership.created_at` 단일 기준 (apps.created_at 분기 폐기). 부수효과: 레거시 앱 없음 → `INNER JOIN ro`, "미배정" 버킷 폐기 |
| b | "호출 수" 부서 기준 | **owner 기준** 일관 (KPI #3, dept-activity 호출, drill-through 모두) |
| c | api_call(nginx) 포함 여부 | **제외**, `message_send` + `workflow_execute` 두 액션만 |

**RBAC JOIN = 옵션 B (사전 JOIN하되 ID만)**: enriched MView에 `app_owner_dept_id`, `actor_dept_id` (ID만) 보존. 부서명(`departments.name`)은 컴포넌트/Layer 2 쿼리에서 query-time JOIN. 부서명 변경 즉시 반영, ownership·membership 변동은 refresh 주기(5분) stale. 옵션 A(이름까지 박음 — 변경 5분 stale)/C(완전 분리 — drill 정확 users 카운트에서 만 단위 dept_members hash build 반복)/D(이중 mat — 비용 2배) 비교 후 채택.

### 2.5.2 채택 vs 호출의 부서 기준 분리

| 컴포넌트 | 의미 | 부서 기준 |
|---|---|---|
| kpi-cards #2 부서별 채택 앱 수 (Breadth) | 우리 부서원이 실제로 쓰는 앱 종류 수 | **actor** |
| kpi-cards #3 API 호출 (Depth) | 우리 부서가 소유한 앱이 받은 호출량 | **owner** |
| kpi-cards #4 Top 앱 / drill-through | 호출량 기반 | **owner** |
| dept-activity 호출/토큰 컬럼 | 부서 소유 앱의 활동량 | **owner** |
| dept-activity 신규 a/k/t 컬럼 | 부서가 새로 보유한 자산 | **owner** (ownership.owner_dept) |

→ Layer 2 mat에 `actor_dept_id`, `app_owner_dept_id` **둘 다 GROUP BY 컬럼**으로 보존. 같은 호출 1건이 두 컴포넌트에서 다른 부서로 분류됨 (마케팅 김대리가 IT챗봇 호출 → actor=MKT / owner=IT). mock 5×5 부서 기준 45K행/월, 운영 N×N 부서 확장 시에도 백만 행 이내.

### 2.5.3 마트 구조 — 2계층

```
[원본]    spx_audit_events (Generated Column 8개 보강)
           + resource_ownership, departments, department_members (회사 RBAC)
              │
              ▼
[Layer 1] spx_mv_audit_enriched               ← MView, 이벤트 1건 = 1행 (event-level)
[Layer 1] spx_v_resource_ownership_enriched  ← View, 자산 ↔ 부서 매핑 (state)
              │
              ▼
[Layer 2] spx_mv_kpi_calls_daily             ← MView, 호출 메트릭 공용 큐브
[Layer 2] spx_mv_model_tokens_daily          ← MView, 모델 코스트 전용 큐브
```

> 모두 `public` schema. 위 다이어그램은 schema 생략 표기.

**객체별 의미**:

| 객체 | 형태 | 한 줄 의미 |
|---|---|---|
| `spx_audit_events` Generated Column 8개 | 컬럼 | details JSONB 속 분기/필터 키를 인덱스 가능 컬럼으로 노출해 H-DASH-01/03 방어를 일관 적용 |
| `spx_mv_audit_enriched` | MView | 감사 이벤트 1건 = 1행. actor·owner 부서 ID, `is_debug`/`is_canonical_call` 플래그 사전 결합된 *정규화 이벤트 스트림* — 모든 호출 메트릭의 단일 진실원 (부서명은 query-time JOIN) |
| `spx_v_resource_ownership_enriched` | View | 현재 시점의 자산(앱/KB/도구) ↔ 소유 부서 매핑 *state 스냅샷* |
| `spx_mv_kpi_calls_daily` | MView | (일 × actor부서ID × owner부서ID × 앱 × app_mode) 단위 호출/사용자/에러/토큰 roll-up 큐브 — 채택은 actor, 호출은 owner가 같은 큐브에서 다른 GROUP BY로 나옴 |
| `spx_mv_model_tokens_daily` | MView | (일 × 모델제공자 × 모델ID) 단위 토큰/호출 합계 — message_send(비챗플로우) + workflow_node_execute(모델노드) UNION. `spx_audit_events` 직접 의존 (enriched 경유 안 함) |

**컴포넌트 → 마트 객체 의존**:

| 컴포넌트 | 사용 객체 | 기준 |
|---|---|---|
| kpi-cards #1 총 오브젝트 | spx_v_resource_ownership_enriched | state |
| kpi-cards #2 채택 폭 | spx_mv_kpi_calls_daily | actor |
| kpi-cards #3 API 호출 | spx_mv_kpi_calls_daily | owner |
| kpi-cards #4 Top 앱 | spx_mv_kpi_calls_daily + apps(name) | owner |
| kpi-drill-through 4구역 | spx_mv_kpi_calls_daily + apps(name) + 정확 users는 spx_mv_audit_enriched | owner |
| dept-objects | spx_v_resource_ownership_enriched | state |
| dept-activity 6컬럼 | spx_mv_kpi_calls_daily + spx_v_resource_ownership_enriched | 호출/토큰=owner, 신규=ownership.created_at |
| model-tokens | spx_mv_model_tokens_daily | — |

**왜 2계층인가**: Layer 1만으로 5개 컴포넌트 다 쓸 수 있지만, kpi-cards + drill-through는 동시 조회라 같은 집계를 카드별로 4번 깎으면 비쌈. Layer 2(daily roll-up mat)로 미리 day 단위까지 굴려두면 카드·차트·표 모두 같은 mat을 다른 GROUP BY로 재사용. Layer 1은 "정규화된 이벤트 스트림", Layer 2는 "비즈니스 큐브" — 역할 분리로 미래 진화 경로(incremental fact / 분석 DB)에서 Layer 2만 갈아끼우면 컴포넌트 코드 무수정.

**왜 Layer 1을 분리하나**: drill-through의 *정확* users 카운트는 daily mat 합산(중복 카운트 상한)이 아니라 `COUNT(DISTINCT actor_id)` 필요 → 행 단위 데이터 다시 봐야 함. Layer 1을 분리하면 행 단위 조회(drill)와 집계 조회(카드)가 같은 사전 JOIN 결과를 공유.

**MView vs View 선택 기준 (3축)**: 원본 크기 / 신선도 허용치 / 조회 빈도. audit_enriched는 9백만 행 + 5분 stale OK + 5개 컴포넌트가 다 씀 → MView. ownership_enriched는 수천 행 + 즉시 반영이 자연스러움 → View.

### 2.5.4 DDL 핵심

**Generated Column 8개** (`spx_audit_events` ALTER, 2026-05-15 적용 완료):

```sql
-- 5/15 결정: _d 접미사 컨벤션(details 추출 표시), total_tokens_d BIGINT, target_app_id UUID(resource_ownership.resource_id JOIN 직접)
ALTER TABLE public.spx_audit_events
  ADD COLUMN app_mode_d        TEXT   GENERATED ALWAYS AS (details->>'appMode')         STORED,
  ADD COLUMN model_provider_d  TEXT   GENERATED ALWAYS AS (details->>'modelProvider')   STORED,
  ADD COLUMN model_id_d        TEXT   GENERATED ALWAYS AS (details->>'modelId')         STORED,
  ADD COLUMN total_tokens_d    BIGINT GENERATED ALWAYS AS (NULLIF(details->>'totalTokens','')::bigint) STORED,
  ADD COLUMN error_d           TEXT   GENERATED ALWAYS AS (details->>'error')           STORED,
  ADD COLUMN invoke_from_d     TEXT   GENERATED ALWAYS AS (details->>'invokeFrom')      STORED,
  ADD COLUMN triggered_from_d  TEXT   GENERATED ALWAYS AS (details->>'triggeredFrom')   STORED,
  ADD COLUMN target_app_id     UUID   GENERATED ALWAYS AS (
    CASE
      WHEN target_type = 'app' THEN target_id::uuid
      WHEN details ? 'appId'   THEN (details->>'appId')::uuid
      ELSE NULL
    END
  ) STORED;
```

> **컨벤션 (5/15 결정)**: details에서 추출한 컬럼은 모두 `_d` 접미사로 출처 자체기록 + 미래 top-level 컬럼과 충돌 방지. `target_app_id`만 예외 — top-level `target_id` + `details.appId` 혼용이라 details 추출 컨벤션 미적용. `total_tokens_d` 타입 BIGINT (workflow_runs.total_tokens=bigint 오버플로 방지). `target_app_id` 타입 UUID (resource_ownership.resource_id가 UUID라 직접 JOIN, collector 13종 모두 UUID `::text` 캐스트로 비-UUID 유입 경로 없음).
>
> **왜 Generated Column인가**: 마트 쿼리에서 `details->>'...'` 캐스팅 반복 없애고 인덱싱 가능 컬럼으로. collector 14종은 details JSONB만 잘 박으면 됨 (직접 컬럼 INSERT 강제 시 spx_audit_events 스키마-collector 강결합 + 새 컬럼마다 14개 수정 필요). `STORED` 사유: `VIRTUAL`은 매 쿼리 계산이라 인덱스 불가. enriched/JSONB index 대안 대비 핵심 이점: spx_audit_events 자체를 모든 소비처(마트 + 보안 + 운영 + 임시 분석)의 공용 인터페이스로 만듦.

**인덱스 5개** (마트 가속용, 2026-05-15 적용 완료):

```sql
CREATE INDEX spx_audit_events_action_mart_idx
  ON public.spx_audit_events (action)
  WHERE action IN ('message_send','workflow_execute');  -- partial, enriched 메인 필터
CREATE INDEX spx_audit_events_tenant_target_app_idx ON public.spx_audit_events (tenant_id, target_app_id);
CREATE INDEX spx_audit_events_tenant_actor_idx     ON public.spx_audit_events (tenant_id, actor_id);
CREATE INDEX spx_audit_events_occurred_action_idx  ON public.spx_audit_events (occurred_at, action);
CREATE INDEX spx_audit_events_app_mode_idx         ON public.spx_audit_events (app_mode_d);
```

> 운영 적용 시 `CONCURRENTLY` 옵션은 트랜잭션 밖이라 prisma migration에서는 사용 불가 → 별도 SQL로 수동 실행. 5/15 적용은 데이터 작은 시점이라 lock 비용 0.

**spx_mv_audit_enriched (Layer 1, MView, 옵션 B — ID만)**:

> Generated Column `_d` 접미사를 enriched view 안에서 마트 친화 alias(`app_mode`/`total_tokens` 등)로 노출 — 컴포넌트 쿼리 단순화. `_d`는 "details 출처" 자체기록 컨벤션이라 인터페이스 노출 시점에서는 불필요.

```sql
CREATE MATERIALIZED VIEW public.spx_mv_audit_enriched AS
SELECT
  ae.id, ae.occurred_at,
  ae.tenant_id::uuid    AS tenant_id,               -- 5/15 보정: TEXT→UUID 역캐스트 (tenant_id는 항상 UUID 형식)
  ae.action, ae.actor_type,
  CASE WHEN ae.actor_type IN ('account','end_user')  -- 5/18 보정: 비-UUID actor 안전 처리
       THEN ae.actor_id::uuid
       ELSE NULL
  END                   AS actor_id,                -- api/system은 비-UUID 문자열이라 NULL
  ae.target_app_id,                                 -- UUID (top-level + details 혼용, 접미사 없음)
  ae.app_mode_d         AS app_mode,                -- alias 권장
  ae.model_provider_d   AS model_provider,
  ae.model_id_d         AS model_id,
  ae.total_tokens_d     AS total_tokens,            -- BIGINT
  ae.invoke_from_d      AS invoke_from,
  ae.triggered_from_d   AS triggered_from,
  ae.error_d            AS error_text,
  ro.owner_department_id AS app_owner_dept_id,     -- ID만 (이름은 query-time JOIN)
  dm.department_id       AS actor_dept_id,          -- ID만
  COALESCE(
    ae.invoke_from_d = 'debugger'
    OR ae.triggered_from_d IN ('debugging','rag-pipeline-debugging'),
    FALSE
  ) AS is_debug,                                    -- 5/15 보정: COALESCE 래핑 (NULL 전파 방지)
  COALESCE(
    ae.app_mode_d <> 'advanced-chat' OR ae.action='message_send',
    FALSE
  ) AS is_canonical_call                            -- 5/15 보정: COALESCE 래핑 (NULL 전파 방지)
FROM public.spx_audit_events ae
JOIN public.spx_resource_ownership ro              -- INNER (결정 a) — 5/19 보정: spx_ prefix
       ON ro.tenant_id=ae.tenant_id::uuid           -- 5/15 보정: TEXT→UUID
      AND ro.resource_type='app'
      AND ro.resource_id=ae.target_app_id
LEFT JOIN public.spx_department_members dm         -- LEFT (end_user/api/system은 부서 없음) — 5/19 보정: spx_ prefix
       ON dm.tenant_id=ae.tenant_id::uuid           -- 5/15 보정: TEXT→UUID
      AND dm.account_id = CASE WHEN ae.actor_type='account' THEN ae.actor_id::uuid END  -- 5/18 보정: dm는 account 한정 + 캐스트 안전
      AND dm.is_active=TRUE
WHERE ae.action IN ('message_send','workflow_execute');   -- 결정 c

-- UNIQUE INDEX (CONCURRENTLY refresh 전제). audit_events.id가 자연 UNIQUE
CREATE UNIQUE INDEX spx_mv_audit_enriched_id_idx
  ON public.spx_mv_audit_enriched (id);

-- 조회 인덱스 (5/20 추가 — 7개 drill endpoint Seq Scan 해소)
CREATE INDEX spx_mv_audit_enriched_tenant_occurred_idx
  ON public.spx_mv_audit_enriched (tenant_id, occurred_at);

-- MView owner = audit_writer (5/15 보정 — REFRESH CONCURRENTLY 권한, Layer 2 작업 중 발견 후 소급)
ALTER MATERIALIZED VIEW public.spx_mv_audit_enriched OWNER TO audit_writer;
```

> **`is_debug` 표현식 확장 (5/15 결정)**: `WorkflowRunTriggeredFrom` enum 7종 중 디버깅 의미 2종 모두 포함. `'debugging'` (Studio "디버그" 실행) + `'rag-pipeline-debugging'` (RAG 파이프라인 디버그). 5/14 코드 기반 재조사에서 발견된 신규 enum 값.
>
> **적용 보정 3건 (Layer 1 실제 적용 결과 + 5/18 운영 안전 보강)**:
> - **5/15 — tenant_id 타입 캐스트**: `spx_audit_events.tenant_id`는 TEXT이고 RBAC 테이블(`resource_ownership`, `department_members`)은 UUID라 JOIN 시 타입 불일치 → JOIN 조건과 SELECT 노출 모두 `::uuid` 명시. `tenant_id`는 actor와 달리 항상 UUID 형식이라 단순 캐스트 안전. 배경: collector 13종이 SELECT 단계에서 `app.tenant_id::text AS tenant_id` 등으로 정규화 (`references/audit-details-spec.md`).
> - **5/15 — COALESCE(..., FALSE)**: `invoke_from_d` / `triggered_from_d` / `app_mode_d`가 NULL일 때 boolean 식이 NULL로 전파되면 Layer 2 `WHERE NOT is_debug AND is_canonical_call` 절에서 의도치 않은 행 누락/포함 발생 → 둘 다 FALSE로 떨어뜨려 H-DASH-03 디버깅 필터 안전성 확보.
> - **5/18 — actor_id 비-UUID 안전 처리**: `actor_type`이 account/end_user/api/system 4종인데 **api/system은 비-UUID 문자열**(예: `'system'`). 단순 `::uuid` 캐스트 시 enriched view refresh가 `invalid input syntax for type uuid: "system"`으로 깨지는 잠복 risk. 5/18 1차 drift 정리(`20260518_fix_enriched_tenant_id_cast`) 시점엔 mock 데이터에 account/end_user만 있어 검증 통과 → 운영 진입 전 미연 차단. **SELECT**는 `CASE WHEN actor_type IN ('account','end_user') THEN ::uuid ELSE NULL` 가드, **department_members JOIN**은 `actor_type='account'` 가드로 dm 매칭 한정. end_user/api/system 행은 dm 매칭 안 되니 LEFT JOIN으로 `actor_dept_id=NULL` → sentinel UUID 매핑("미배정") 자연 흐름.

> **INNER vs LEFT 의미**: `resource_ownership` INNER = "매칭되어야 정상"(결정 a) → 매칭 안 되면 행 사라짐 → 모니터링 #3가 캐치 → **사고를 드러나게 하는 장치**. `department_members` LEFT = "end_user/api/system은 매칭 안 되는 게 자연" → 외부 액터 호출도 owner 기준 집계에서 누락 안 됨.
>
> **플래그 컬럼화의 진짜 이유**: 컴포넌트가 N명 × N시간 추가됨. `is_debug` / `is_canonical_call`을 마트 인터페이스에 흡수 안 하면 3개월 후 누군가 H-DASH-01 모르고 SQL 추가하며 이중카운트. "비즈니스 규칙은 마트 안에 흡수하고 컴포넌트는 그냥 SELECT한다"가 마트 설계 황금 규칙. 인덱스 가능 + 자기문서화 + 정의 한 곳에서 변경.

**spx_mv_kpi_calls_daily (Layer 2, 옵션 B)**:

```sql
CREATE MATERIALIZED VIEW public.spx_mv_kpi_calls_daily AS
SELECT
  tenant_id,
  date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul') AS day,
  COALESCE(app_owner_dept_id, '00000000-0000-0000-0000-000000000000'::uuid) AS app_owner_dept_id,  -- 5/15 보정: NULL → sentinel UUID
  COALESCE(actor_dept_id,     '00000000-0000-0000-0000-000000000000'::uuid) AS actor_dept_id,
  target_app_id, app_mode,
  COUNT(*) AS calls,
  COUNT(DISTINCT actor_id) AS users,
  COUNT(*) FILTER (WHERE error_text IS NOT NULL) AS errors,
  SUM(total_tokens) AS tokens,
  MAX(occurred_at) AS last_seen
FROM public.spx_mv_audit_enriched
WHERE NOT is_debug AND is_canonical_call
GROUP BY 1, 2, 3, 4, 5, 6;
-- 컴포넌트 쿼리에서 sentinel UUID = '미배정'으로 매핑

-- UNIQUE INDEX (CONCURRENTLY refresh 전제, 단순 컬럼만 — 표현식 인덱스 X)
CREATE UNIQUE INDEX spx_mv_kpi_calls_daily_unique_idx
  ON public.spx_mv_kpi_calls_daily
     (tenant_id, day, app_owner_dept_id, actor_dept_id, target_app_id, app_mode);

-- MView owner = audit_writer (REFRESH MATERIALIZED VIEW CONCURRENTLY는 owner만 실행 가능)
ALTER MATERIALIZED VIEW public.spx_mv_kpi_calls_daily OWNER TO audit_writer;
```

타임존 = `Asia/Seoul` day 경계 (사용자 인지와 일치). 기간 선택기가 day 경계만 받음 → 모든 기간 비교는 `SUM` 한 번.

> **5/15 적용 보정 (Layer 2 실제 적용 결과)**:
> - **CONCURRENTLY UNIQUE INDEX 표현식 제약**: PostgreSQL은 표현식 인덱스(`COALESCE(...)` in INDEX)로 `REFRESH MATERIALIZED VIEW CONCURRENTLY` 불가. 해결 = SELECT 단계에서 sentinel UUID로 NULL 치환 → INDEX는 단순 컬럼. 컴포넌트 쿼리는 `'00000000-...'` UUID를 "미배정"으로 매핑.
> - **MView OWNER 필수**: `REFRESH MATERIALIZED VIEW CONCURRENTLY`는 MView owner만 실행 가능. collector worker가 audit_writer로 접속해 refresh하므로 OWNER도 audit_writer. 신규 MView 생성 시 항상 `ALTER ... OWNER TO audit_writer` 동반 필수.

**spx_mv_model_tokens_daily (Layer 2 — B안 UNION 재작성, 2026-06-10)**:

> **의존 변경**: `spx_mv_audit_enriched` → **`spx_audit_events` 직접 조회** (모델 차트는 부서 차원 없어 RBAC JOIN 불필요). enriched 의존 제거로 refresh chain에서 독립(§ 2.5.5 참조).

```sql
-- Partial index for workflow_node_execute model rows (REFRESH scan acceleration)
CREATE INDEX IF NOT EXISTS spx_audit_events_wf_model_idx
  ON public.spx_audit_events (occurred_at, model_id_d)
  WHERE action = 'workflow_node_execute' AND model_id_d IS NOT NULL;

CREATE MATERIALIZED VIEW public.spx_mv_model_tokens_daily AS
SELECT
  tenant_id,
  date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul') AS day,
  COALESCE(model_provider_d, '미분류') AS model_provider,
  COALESCE(model_id_d, '미분류')       AS model_id,
  SUM(total_tokens_d)                  AS tokens,
  COUNT(*)                             AS calls
FROM (
  -- (1) Non-chatflow messages (chat / completion / agent-chat)
  SELECT tenant_id, occurred_at, model_provider_d, model_id_d, total_tokens_d
  FROM public.spx_audit_events
  WHERE action = 'message_send'
    AND app_mode_d <> 'advanced-chat'
    AND model_id_d IS NOT NULL
    AND total_tokens_d IS NOT NULL
    AND NOT COALESCE(
          invoke_from_d = 'debugger'
          OR triggered_from_d IN ('debugging','rag-pipeline-debugging'),
          FALSE)

  UNION ALL

  -- (2) Workflow + chatflow node executions (model-bearing nodes only)
  SELECT tenant_id, occurred_at, model_provider_d, model_id_d, total_tokens_d
  FROM public.spx_audit_events
  WHERE action = 'workflow_node_execute'
    AND app_mode_d IN ('workflow', 'advanced-chat')
    AND model_id_d IS NOT NULL
    AND total_tokens_d IS NOT NULL
    AND NOT COALESCE(
          invoke_from_d = 'debugger'
          OR triggered_from_d IN ('debugging','rag-pipeline-debugging'),
          FALSE)
) sub
GROUP BY 1, 2, 3, 4;

-- UNIQUE INDEX (CONCURRENTLY refresh 전제). model_provider/model_id는 SELECT에서 COALESCE로 NOT NULL 보장
CREATE UNIQUE INDEX spx_mv_model_tokens_daily_unique_idx
  ON public.spx_mv_model_tokens_daily (tenant_id, day, model_provider, model_id);

-- MView owner = audit_writer (REFRESH CONCURRENTLY 권한)
ALTER MATERIALIZED VIEW public.spx_mv_model_tokens_daily OWNER TO audit_writer;
```

> **이중집계 차단 (H-DASH-01)**: 챗플로우(`advanced-chat`) message_send는 (1)에서 제외. 챗플로우 토큰은 (2)의 노드 실행에서만 집계. 비챗플로우 메시지는 기존대로 (1)에서 집계.
>
> **H-DASH-11 방어**: `process_data` JSON 직접 파싱 없음. 모든 필드는 generated column(`_d` 접미사)에서 추출. collector가 1회 파싱 → `details` JSONB → generated column 자동 추출 경로.

`spx_mv_kpi_calls_daily`와 별도 mat 사유: 그룹핑 차원이 완전히 다름 (calls는 부서×앱 / tokens는 모델). 합치면 카디널리티 폭증.

**spx_v_resource_ownership_enriched (Layer 1, View)**:

```sql
CREATE VIEW public.spx_v_resource_ownership_enriched AS
SELECT
  ro.tenant_id, ro.resource_type, ro.resource_id,
  ro.owner_department_id, d.name AS owner_dept_name,
  ro.owner_account_id, ro.created_at
FROM public.spx_resource_ownership ro               -- 5/19 보정: spx_ prefix
JOIN public.spx_departments d ON d.id=ro.owner_department_id;  -- 5/19 보정: spx_ prefix

-- View owner = audit_writer (5/15 보정 — 일관성 위해 모든 마트 객체 owner 통일)
ALTER VIEW public.spx_v_resource_ownership_enriched OWNER TO audit_writer;
```

수백~수천 행 규모라 View로도 풀스캔 비용 무시. RBAC 변경 즉시 반영 우선.

**컴포넌트 쿼리 예 (부서명 query-time JOIN)**:

```sql
SELECT d.name AS dept, SUM(c.calls) AS calls, SUM(c.tokens) AS tokens
FROM public.spx_mv_kpi_calls_daily c
JOIN public.departments d ON d.id = c.app_owner_dept_id
WHERE c.tenant_id=:tid AND c.day BETWEEN :start AND :end
GROUP BY d.name;
```

### 2.5.5 새로고침 + 모니터링

**Collector trigger 연동 — 5분 polling cycle 끝에 chain refresh** (2026-05-15 결정. pg_cron 미채택):

dify-audit 컨테이너의 db-poller (5분 주기)가 모든 collector run 완료 후 마지막 단계로 마트 chain refresh를 직접 호출. 별도 스케줄러(pg_cron, systemd cron) 없음.

```typescript
// dify-audit/src/workers/db-poller.ts (개념 윤곽)
async function refreshMartChain() {
  try {
    // enriched → kpi_calls_daily: 의존 순서 유지 (kpi_calls는 enriched 기반)
    await prisma.$executeRawUnsafe(
      `REFRESH MATERIALIZED VIEW CONCURRENTLY public.spx_mv_audit_enriched`
    );
    await prisma.$executeRawUnsafe(
      `REFRESH MATERIALIZED VIEW CONCURRENTLY public.spx_mv_kpi_calls_daily`
    );
    // model_tokens_daily: spx_audit_events 직접 의존 (enriched 불필요)
    // enriched와 병렬 가능하나, 순차 유지 (단순성 + refresh 부하 분산)
    await prisma.$executeRawUnsafe(
      `REFRESH MATERIALIZED VIEW CONCURRENTLY public.spx_mv_model_tokens_daily`
    );
    logger.info('[mart] chain refresh completed');
  } catch (err) {
    logger.error('[mart] chain refresh failed', err);
    // 마트 실패가 collector polling cycle 다음 회차를 차단하면 안 됨
  }
}

// db-poller run 끝에서 호출
await runAllCollectors();
await refreshMartChain();
```

> **왜 collector trigger인가 (vs pg_cron)**: 우리 환경 `postgres:15-alpine`은 pg_cron 미포함 → image 교체 또는 Dockerfile build 필요. 인프라 부담 추가. 반면 collector trigger는 코드 5줄로 끝. **collector polling이 5분 주기인데 마트 cron이 별도로 도는 의미가 약함** — collector worker 죽으면 신규 데이터도 안 들어오고 마트 refresh도 의미 없음. 둘이 같이 가는 게 자연스러움.
>
> **왜 chain인가**: enriched → kpi_calls_daily는 의존 순서 보장 필수. **model_tokens_daily는 B안(2026-06-10) 이후 `spx_audit_events` 직접 의존으로 enriched와 독립**이나, 순차 호출 유지 (단순성 + DB I/O 분산). 병렬 최적화는 운영 진입 후 refresh 소요가 병목이 될 때 검토.
>
> **왜 5분 주기 + CONCURRENTLY**: collector polling이 5분이라 마트도 자연스럽게 5분. TanStack staleTime 5분과 정합. `CONCURRENTLY`로 refresh 중에도 SELECT 가능 (전제: UNIQUE INDEX).
>
> **실시간 옵션 미채택 사유**: 전체 lag = MAX(병목 단계). collector 5분 폴링이 진짜 병목이라 마트만 실시간(pg_trigger) 만들어도 사용자 체감 0. 진짜 실시간 필요 시 collector도 함께 실시간화 (§ 2.5.6 단계 B/C).
>
> **운영 진입 시 재검토 가치**: 책임 분리(collector vs mart) + 모니터링 메트릭 #1a/#1b 분리 진단 가치가 커지면 pg_cron으로 마이그레이션 검토. 현재는 개발 단계 빠른 진행 우선.

**모니터링 메트릭 6개** — collector trigger 연동으로 #1b/#2 정의 변경:

| # | 메트릭 | 임계 |
|---|---|---|
| 1a | **collector lag**: `now() - max(occurred_at) FROM spx_audit_events` | > 10분 |
| 1b | **mart lag**: `max(occurred_at FROM spx_audit_events) - max(occurred_at FROM spx_mv_kpi_calls_daily)` | > 7분 |
| 2 | chain REFRESH 소요 (collector 로그 `[mart] chain refresh completed` 직전 시각 - `[db-poller] Run completed` 시각 차) | > 4분 |
| 3 | enriched vs src 행수 차이 (대상 action 한정) | > 0 → ownership 누락 사고 |
| 4 | `resource_ownership.owner_department_id IS NULL` 카운트 | > 0 → 부서 누락 사고 |
| 5 | enriched의 actor_dept_id IS NULL & actor_type='account' 비율 | > 5% → department_members 동기화 사고 |

> **#1b가 의미 약해진 점**: collector trigger 연동이라 mart lag = chain refresh 소요만 반영 (사실상 #2와 거의 동일). pg_cron 시절엔 collector run과 cron tick이 어긋나는 만큼 #1b가 별도 의미였음. 운영 진입 후 pg_cron으로 마이그레이션하면 #1b 재의미화.

> **3단계 사고 캐치 메타**: #3은 적재 단계(target_app_id가 RBAC ownership에 미등록), #4는 등록 단계(ownership에 등록됐는데 부서 NULL), #5는 동기화 단계(자산 OK인데 사용자 부서 매핑 누락). 마트 객체는 사고 나도 묵묵히 돌아감 → 메트릭이 옆에서 watching해서 *마트가 거짓말하는 순간*을 감지.
>
> **RBAC 변경 stale의 두 시나리오**: 인사이동(dept_members 변경)은 refresh 대기 5분. 부서명 변경(departments.name)은 DB 시점 0초(옵션 B의 실질 이점), 프론트 캐시만 5분. ⚠️ 인사이동 시 한계 — refresh 후 김대리가 IT 시절 한 과거 호출도 MKT로 재분류됨(SCD type 2 아님).

### 2.5.6 실시간 확장 경로

```
[현재]  Postgres pg_cron 5분 (배치 MView)
   ↓ 부하 증가 / lag 5분 부족
[A]    각 mat에 incremental refresh (last_refreshed_at 컬럼)
   ↓ 실시간성 요구 (30초 이내 반영)
[B]    PG logical replication / pg_trigger → Redis Stream
            → 별도 fact 테이블 incremental upsert
   ↓ 글로벌 / 멀티테넌트 스케일
[C]    분석 DB 분리 (ClickHouse / TimescaleDB)
```

핵심: Layer 2가 컴포넌트의 인터페이스 — `spx_mv_kpi_calls_daily`만 보면 아래 구현이 MView → incremental fact → 분석DB로 바뀌어도 컴포넌트 코드 무수정.

### 2.5.7 구축 순서

0. ✅ **`audit_events` 테이블 rename → `spx_audit_events`** (2026-05-15 완료) — dify-audit collector INSERT target rename + Prisma schema rename + trigger 함수 `public.spx_log_dify_change()` 재배포 + audit schema `DROP CASCADE`
1. ✅ **collector P0 보강** (2026-05-15 완료) — 4 collector patch diff ~14줄 적용 (`references/audit-details-spec.md § P0 patch diff A.1~A.4`)
2. ✅ **`spx_audit_events` Generated Column 8개 + 인덱스 5개** (2026-05-15 완료) — `_d` 접미사 컨벤션, total_tokens_d BIGINT, target_app_id UUID
3. ✅ **Layer 1: `spx_mv_audit_enriched` MView + `spx_v_resource_ownership_enriched` View** (2026-05-15 완료) — UNIQUE INDEX `spx_mv_audit_enriched_id_idx` 포함. 보정 2건 (`::uuid` 캐스트 + `COALESCE` 래핑) 적용
4. ✅ **Layer 2: `spx_mv_kpi_calls_daily`, `spx_mv_model_tokens_daily`** (2026-05-15 완료) — UNIQUE INDEX + ALTER OWNER TO audit_writer + COALESCE sentinel UUID(`00000000-...`) 보정 반영. 검증: Layer 1 292 = Layer 2 calls 합 292 완전 일치
5. ✅ **Collector trigger 연동 chain refresh + 모니터링 메트릭** (2026-05-15 완료) — pg_cron 미채택, db-poller polling cycle 끝에 `refreshMartChain()` 호출 (+16줄, try/catch 격리). 로그 `[mart] chain refresh completed` 박힘
6. ✅ **백엔드 service 재작성** (2026-05-18 완료) — 컴포넌트 5종 + drill-through 11종 모두 Layer 2/enriched view로 전환. grep 검증 4종 통과 (30-1/30-2/30-3 = 0 hit, 30-4 = 22 hits). KPI 4종 구성 반영 (#1 총 오브젝트 / #2 채택 앱 수(actor) / #3 API 호출(owner) / #4 앱별 통계). sentinel UUID → "미배정" 서비스 응답 직전 매핑. mart.py 모델 4종 신설 (`__table_args__ = {'info': {'is_view': True}}`)
7. ⏳ **부하 테스트** ← **현재 위치 (다음 작업)** — p95 응답 3초 임계 검증. EXPLAIN ANALYZE 인덱스 hit 확인. mart lag / chain refresh 소요 모니터링 메트릭 검증

> 사람용 통합 시나리오 (호출 1건이 KPI #3 카드까지 가는 end-to-end 흐름 + 사고 차단 3지점): [[0. Inbox/마트 설계 결정 - 2026-05-14.md|마트 설계 결정 노트 § 8]] 참조.

## 3. 데이터 흐름: 화면 → API → 테이블

### KPI 카드 4종

```
[총 오브젝트]
  API: GET /admin/dashboard/kpi
  쿼리: spx_resource_ownership → COUNT GROUP BY resource_type
  방어: H-DASH-04 (미배정 앱 fallback)

[활성 사용자]
  API: (위와 같은 엔드포인트)
  쿼리: messages.from_end_user_id DISTINCT + workflow_runs.created_by DISTINCT
  방어: H-DASH-08 (NULL 제외)

[API 호출]
  API: (위와 같은 엔드포인트)
  쿼리: messages COUNT + workflow_runs COUNT
  방어: H-DASH-01 (ADVANCED_CHAT 이중카운트), H-DASH-03 (디버깅 제외)

[토큰 사용]
  API: (위와 같은 엔드포인트)
  쿼리: messages.(message_tokens + answer_tokens) SUM + workflow_runs.total_tokens SUM
  방어: H-DASH-01 (ADVANCED_CHAT 이중카운트), H-DASH-03 (디버깅 제외)

[공통]
  증감률: 현재 기간 vs 이전 기간 → 쿼리 2회
  방어: H-DASH-05 (이전 기간 데이터 삭제), H-DASH-07 (분모 0)
```

### 부서별 오브젝트 분포 (스택 막대 차트)

```
  API: GET /admin/dashboard/dept-objects
  쿼리: spx_departments → spx_resource_ownership → COUNT GROUP BY (dept, resource_type)
  방어: H-DASH-04 (미배정 → "미배정" 부서 fallback)
  캐시: staleTime 5분
```

### 모델별 토큰 사용량 (수평 막대 차트)

```
  API: GET /admin/dashboard/model-tokens
  쿼리:
    CHAT 계열: messages → GROUP BY (model_provider, model_id) → 토큰 SUM
    WORKFLOW: workflow_runs → total_tokens 합산 → "미분류" 버킷
  방어: H-DASH-01 (이중카운트), H-DASH-02 (WORKFLOW 모델 분리 불가), H-DASH-09 (로컬 라벨)
  캐시: staleTime 5분
  선택적 2차: workflow_node_executions JSON 파싱 → H-DASH-11 (성능) 주의
```

### 부서별 활동 테이블

```
  API: GET /admin/dashboard/dept-activity
  쿼리: spx_departments → spx_resource_ownership → apps →
        messages (COMPLETION/CHAT/AGENT_CHAT/ADVANCED_CHAT)
        + workflow_runs (WORKFLOW)
        GROUP BY department
  컬럼: 부서명, 신규 App, 신규 KB, 신규 Tool, 호출 수, 토큰 사용
  방어: H-DASH-01 (이중카운트), H-DASH-03 (디버깅 제외),
        H-DASH-04 (미배정), H-DASH-10 (다단 JOIN 성능)
  캐시: Redis TTL 5분 권장 (가장 무거운 쿼리)
```

> ※ CSV 내보내기는 화면 설계에 없어 제외 (2026-04-30 결정). H-DASH-12는 향후 도입 시 재적용.

## 4. AppMode별 쿼리 분기 규칙

> 이 규칙은 KPI·부서별 활동·모델별 토큰 등 **모든 집계 쿼리에 공통 적용**.

| AppMode | 토큰 조회 테이블 | 필터 조건 | 비고 |
|---------|:-------------:|----------|------|
| COMPLETION | `messages` | `invoke_from != 'debugger'` | |
| CHAT | `messages` | `invoke_from != 'debugger'` | |
| AGENT_CHAT | `messages` | `invoke_from != 'debugger'` | |
| ADVANCED_CHAT | `messages`만 | `invoke_from != 'debugger'` | workflow_runs에도 같은 값 있지만 **messages만 읽어서 이중카운트 방지** (H-DASH-01) |
| WORKFLOW | `workflow_runs` | `triggered_from = 'app-run'` | messages에 저장 안 됨 |

## 5. 변경 영향 규칙 (Change Rules)

프론트매터에도 선언했지만, 여기서 상세하게 기술.

### Rule 1: RBAC 테이블 스키마 변경

```
트리거: 김이사님이 spx_departments / spx_resource_ownership / spx_department_members 등 RBAC 테이블 컬럼 변경 (5/4 가정명 sp_users/sp_user_departments는 spx_accounts + spx_department_members로 귀결)
영향 범위:
  - SQLAlchemy 모델 클래스 (models/rbac.py 또는 해당 파일)
  - 모든 집계 쿼리 (KPI, 부서별 오브젝트, 부서별 활동)
  - 부서별 사용자 수 쿼리
조치:
  1. 모델 클래스 필드명 수정
  2. 영향받는 쿼리/서비스 함수 목록 확인 (모델 클래스 사용처 검색)
  3. 테스트 재실행
관련 Harness: H-DASH-13
```

### Rule 2: AppMode별 토큰 저장 위치 변경

```
트리거: Dify 업데이트로 토큰 저장 테이블/필드가 바뀜 (가능성 낮지만 존재)
영향 범위:
  - KPI 토큰 합산 로직
  - 부서별 활동 테이블 토큰 컬럼
  - 모델별 토큰 차트
조치:
  1. AppMode별 쿼리 분기 규칙 (섹션 4) 재검토
  2. Defect Catalog H-DASH-01 방어 로직 재확인
관련 Harness: H-DASH-01
```

### ~~Rule 3: Keycloak upsert 매핑 키 변경~~ → 폐기 (2026-05-06 자동 해소)

`accounts.id` 직접 매핑이 회사 표준으로 확정되며 별도 upsert 매핑 키 불필요. H-DASH-14 자동 해소.

### ~~Rule 4: 설정 모달 사이드바 구조 합의 변경~~ → 폐기 (2026-05-13 라우트 이동)

대시보드가 `/dashboard` 톱레벨 라우트로 이동하면서 설정 모달 사이드바 의존 자체 소멸. H-DASH-15 종결.

### Rule 5: audit collector 보강 (신규 — 2026-05-13)

```
트리거: ✅ P0 collector 보강 완료 (사용자 본인 트랙, 2026-05-15 — appMode × 4 collector + triggeredFrom × workflow-runs.ts)
영향 범위:
  - KPI 카드 (API 호출 / 토큰 / 에러)
  - 부서별 호출/에러 drill-through
  - AppMode 분기 방어 (H-DASH-01)
  - 디버깅 필터 방어 (H-DASH-03)
조치 (모두 완료):
  1. ✅ audit details 필드 보강 확인 (`references/audit-details-spec.md § P0 patch diff A.1~A.4`)
  2. ✅ Generated Column `app_mode_d`로 노출 (마트 ETL 쿼리는 enriched view의 `app_mode` alias 사용)
  3. ✅ `is_debug` 플래그에 `triggered_from_d IN ('debugging','rag-pipeline-debugging')` 흡수
관련 Harness: H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing (해소됨)
```

## 6. 구현 순서 가이드

외부 의존을 고려한 현실적 순서.

```
[즉시 착수 가능]
  ① 설정 모달 사이드바 협의 → 구조 선점
  ② 백엔드 API 뼈대 (Blueprint 등록, 엔드포인트 라우트)
  ③ 프론트엔드 페이지 뼈대 (라우트, 레이아웃, 목데이터)

[RBAC 테이블 스키마 확정 후]
  ④ SQLAlchemy 모델 클래스 작성
  ⑤ 집계 쿼리 구현 (KPI, 부서별 오브젝트, 모델별 토큰)
  ⑥ 부서별 활동 쿼리 (가장 복잡)

[Keycloak upsert 완료 후]
  ⑦ 사용자↔부서 매핑 쿼리 추가
  ⑧ 활성 사용자 KPI 부서 기반으로 전환

[전체 연동]
  ⑨ 프론트↔백 연결, 실제 데이터 확인
  ⑩ Gate Review (Harness 테스트)
```

## 6.5. 스타일 가이드 (전역 공통)

> 모든 컴포넌트가 따라야 할 시각/포맷 규칙. 신규 컴포넌트 작성 시 **반드시 참조**.
> 상세: [[4. 지식노트/Dify - 디자인 토큰과 차트·카드 색상 패턴.md]]

### 카드 컨테이너 표준 (2026-05-14 갱신 — 라우트 페이지 환경 + 2분리)

대시보드가 모달 → `/dashboard` 라우트 페이지로 이동(2026-05-13)한 뒤 첫 실측 결과, 모달용 단일 카드 패턴이 페이지 환경에선 시각적 무게가 어색했음. 스튜디오(`/apps`) / 지식(`/datasets`) 페이지의 카드 톤에 맞춰 **2종 분리**:

| 카드 종류 | 적용 대상 | 참조 패턴 |
|---|---|---|
| **메인 카드 (KPI 카드)** | KPI 4종 카드 (kpi-section) | 스튜디오 앱 카드 (`AppCard`) — hover 상승감 있는 콘텐츠 카드 톤 |
| **컨테이너 카드 (차트·표)** | dept-objects / model-tokens / dept-activity / drill 차트·표 | 스튜디오 "앱 만들기" 사이드 카드 (`NewAppCard`) — 평평한 메뉴/섹션 톤 |

> 정확한 className은 `web/app/(commonLayout)/apps/Apps.tsx` 및 관련 컴포넌트에서 추출하여 사용. raw 색상값 금지(토큰만), 다크/라이트 자동 대응 유지.

**이전 단일 패턴** `rounded-xl border-[0.5px] border-divider-regular bg-components-card-bg px-6 py-4 shadow-xs` (2026-05-04 ~ 2026-05-13)는 폐기.

**결정 근거**: 모달 환경에선 모든 영역이 PROVIDER 탭 톤으로 통일된 게 자연스러웠으나, 라우트 페이지에선 KPI(메인 정보) vs 차트·표(컨테이너 섹션)의 위계가 시각적으로도 드러나야 함. 스튜디오 페이지가 같은 commonLayout 안에서 이미 그 위계를 표현하고 있어, 동일 패턴을 차용하여 톤 일관성 확보.

**Phase 1 5컴포넌트 + Phase 2 영향 범위**:
- `kpi-section/kpi-card.tsx` 카드 컨테이너
- `kpi-section/index.tsx` 로딩/에러/빈 상태 자리표시자
- `dept-objects-chart/index.tsx` 4개 상태
- `model-tokens-chart/index.tsx` 4개 상태
- `dept-activity-table/index.tsx` 4개 상태
- `chart-drawer/summary-cards.tsx` 요약 카드 3종 (Phase 2 spec)

**구현 가이드**:
- 5개 컴포넌트가 동일 className을 반복하므로 공통 컨테이너 컴포넌트 신설 검토:
  - 위치 후보: `web/app/components/admin/shared/dashboard-card.tsx`
  - props: `className?`, `children`, (선택) `padding?: 'normal' | 'compact'`
  - 단순 wrapper라 컴포넌트 spec까진 안 만들고 conventions.md에 패턴만 명시

### 색상 매핑 (오브젝트 타입 = 부서별 차트와 통일)

| 항목 | 토큰 | 용도 |
|------|------|------|
| App | `util-colors-blue-blue-500` | 앱/기능 |
| KB | `util-colors-teal-teal-500` | 지식 베이스 |
| Tool | `util-colors-orange-orange-500` | 도구 |
| 미배정 | `text-text-tertiary` | fallback |

> ※ blue/teal 구분이 약할 수 있음 — 추후 재조정 가능. 지금은 임시 결정.

### 색상 매핑 (시맨틱)

| 의미 | 토큰 | 용도 |
|------|------|------|
| 상승 / 정상 | `text-text-success` (#079455) | 증감 ▲, 활성 상태 |
| 하락 / 위험 | `text-text-destructive` (#d92d20) | 증감 ▼, 에러 |
| 강조 | `text-text-accent` (#155aef) | 링크, 활성 탭 |
| 보조 | `text-text-tertiary` | 0%, N/A, 부가 정보 숫자 |

### 숫자 포맷 (전역)

- `< 10,000` → raw + 천단위 콤마 (`9,820`)
- `≥ 10,000` → K/M 압축 1자리 소수 (`15.4K`, `2.8M`)
- 압축 표시 시 **hover 툴팁에 정확값** 필수

### 기간 표시

- 페이지 헤더에서 선택한 라벨을 각 카드/차트의 제목 밑에 작은 회색으로 표시
- 형식: `"지난 7 일"` (Dify와 동일하게 띄어쓰기)

### 화면 설계 이미지 참조

- spec 작성 시 frontmatter `design_image:` 필드에 경로 명시
- 구현 시 spec과 함께 이미지를 반드시 참조 (1:1 대조)

### 화면 레이아웃 — 고정 높이 + 스크롤 (2026-05-13 신규 전역 정책)

- 카드 / 차트 / 표 모두 **고정 높이**
- 콘텐츠 초과 시 **영역 내부 스크롤** (외부로 빠지지 않음)
- 데이터 양에 따라 카드 크기 변동되는 레이아웃 **금지**
- 표는 헤더 고정 + 본문 스크롤
- **본문 고정 높이값 (2026-06-09 확정)**: `max-h-*` 대신 고정 `h-*` — 드릴 차트 `h-[250px]` / 표 `h-[392px]` / 모델별 토큰 `h-[260px]` / 부서별 오브젝트 `h-[280px]`. 빈/에러 상태도 **데이터와 같은 높이**. (스크롤 임계/모바일 반응형은 추후)
- **빈 상태에도 제목/축 골격 유지** (위젯 통째 early-return 금지), 결측치 `-`, 부서는 전수 표시 → 상세는 `conventions.md § 빈 상태 / 골격 / 결측치 정책` (SoT)

### 인터랙션 정책 (2026-05-13)

- **좌하 차트 클릭**: 동작 없음 (차트 드로어 보류로 트리거 제거). 차트만 표시
- **표 행 클릭**: KPI 4번(앱별 통계)만 → 같은 탭 Dify 모니터링 페이지(`/app/{appId}/overview`). KPI 1·2·3 표 클릭은 없음
- **KPI 카드 클릭**: drill-through 펼침 (Phase 2)
- **ESC 키**: 자유 활용 (drill-through 해제 등 — 라우트 페이지 환경)
- **표 헤더 정렬/필터**: 후보 (전역 정책 결정 시 모든 표에 일관 적용)

## 7. Harness 매핑 요약

| 화면 요소 | 적용할 Harness |
|-----------|---------------|
| 대시보드 컨트롤 (기간 선택 + 새로고침) | (없음 — 데이터 집계 없음) |
| KPI 1: 총 오브젝트 | H-DASH-04 |
| KPI 2: 부서별 채택 앱 수 | H-DASH-04 (미배정 fallback), H-DASH-08 (NULL actor 제외) |
| KPI 3: API 호출 | H-DASH-01, H-DASH-03, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing |
| KPI 4: 앱별 통계 (Top 앱 호출 + 앱별 호출 막대 + Top 에러 앱 + 6컬럼 표) | H-DASH-01, H-DASH-03, H-DASH-18 (ILIKE 분류) |
| KPI 증감률 (공통) | H-DASH-05, H-DASH-07 |
| 부서별 오브젝트 차트 | H-DASH-04 (resource_ownership 직접 + apps LEFT JOIN 패턴 권장) |
| 모델별 토큰 차트 | H-DASH-01, H-DASH-02, H-DASH-09 (워크플로 모델 정보 누락 — P1 선택 보강) |
| 차트 드로어 4종 (보류) | H-DASH-20 (보류 함정 참조) |
| 부서별 활동 테이블 | H-DASH-01, H-DASH-03, H-DASH-04, H-DASH-10 |
| 설정 모달 사이드바 | H-DASH-15 |
| 전체 (비동기 데이터) | H-DASH-06 |
| 전체 (RBAC 연동) | H-DASH-13 |
| 전체 (사용자 매핑) | H-DASH-14 |

## 10. 미해결 결정

> 마트 골격 변경 없이 옵션 토글로 처리 가능한 항목 + 외부 의존 / PM 확인 필요 항목 모음.

### 10.1 보류된 미세 결정 (마트 골격 무영향)

| # | 항목 | 현재 권장 | 재판단 트리거 |
|---|---|---|---|
| d | drill-through users 정확도 | 카드/표 헤드=daily 합산(근사), drill 표만 enriched 직조회 `COUNT DISTINCT` | drill 응답 p95 > 3초 또는 정확도 클레임 발생 |
| e | Read Replica 분리 | 같은 dify Postgres로 시작 (일 호출 < 10만 가정) | 부하 테스트 p95 > 3초 또는 mart_refresh가 OLTP에 영향 |
| f | 타임존 | KST day 경계 확정 (§ 2.5.4 DDL 반영) | 멀티 리전 진입 시 |
| g | dbt 도입 | Phase 1 객체 6개는 직접 마이그레이션, 객체 ≥ 10개 시점에 도입 검토 | 마트 객체 추가 누적 시 |

### 10.2 외부 의존 — collector P0 보강

> 마트 작동 차단 요소. 본 작업은 사용자 본인 트랙(5/14 우선순위 2)으로 진행 중.

| 보강 대상 | 추가 키 | 영향 Generated Column | 방어 Harness |
|---|---|---|---|
| `messages.ts` | `appMode` | app_mode | H-DASH-01 |
| `workflow-runs.ts` | `appMode`, `triggeredFrom` | app_mode, triggered_from | H-DASH-01, H-DASH-03 |
| `workflow-nodes.ts` | `appMode` | app_mode | H-DASH-01 |
| `conversations.ts` | `appMode` | app_mode | H-DASH-01 |

P0 완료 시점부터 신규 INSERT가 Generated Column 자동 채움 → 마트 풀가동. 명세: `references/audit-details-spec.md § P0 collector 보강 명세`.

### 10.3 PM 확인 필요 (마트 정합성 영향)

| # | 항목 | 영향 | 확인 후 반영 |
|---|---|---|---|
| 1 | dept-activity "신규" 정의 | 결정 a에 따라 `resource_ownership.created_at` 단일 기준 박았으나 PM 의도 확정 필요 (자산 등록일 vs 사용 시작일) | 의도가 다르면 mv_kpi_calls_daily에 first_seen 차원 추가 검토 |
| 2 | api_call(nginx) 부서별 분류 누락 수용 여부 | 결정 c로 마트에서 제외했으나 PM이 "총 API 호출에 api_call 포함" 요구 시 | URI → 앱 토큰 → 앱 → 부서 역추적 시스템 별도 트랙 (nginx bearer token 미로깅 한계로 collector 보강 만으로는 해결 불가) |
| 3 | dataset_operator 권한 사용 빈도 | 대시보드 가시성 정책에 영향 (전체 공개 + dataset_operator 잠정 허용 정책 유지/철회) | 별도 가드 추가 시 헤더 `DashboardNav` 노출 조건 + `RoleRouteGuard` 갱신 |

## 관련 노트

- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]] — 메인 프로젝트 노트
- [[3. 프로젝트/SPX-Agent 하네스 설계.md|하네스 설계]] — HDD 작업 계획
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]] — 결함 카탈로그
- [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]]
- [[4. 지식노트/Dify - 통계·토큰 DB 스키마 구조.md]]
- [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
- [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]]
- [[4. 지식노트/Dify - workflow_node_executions에서 모델별 토큰 추출.md]]
