---
tags: [프로젝트, dify, AI-Agent, HDD, infra]
type: spec/design
screen: 데이터 마트 (인프라)
harness: []
date: 2026-05-07
last_updated: 2026-05-15
status: 2026-05-15 rename + collector P0 + Generated Column 8개(_d 접미사) + 인덱스 5개 적용 완료. 현재 Layer 1 진입 직전
---
# 데이터 마트 — Design

> **2026-05-15 진입 차단 결정 4건 적용 완료**: Generated Column `_d` 접미사 / `target_app_id` UUID / `is_debug`에 `rag-pipeline-debugging` 추가 / collector 4건 `app.mode` LEFT JOIN 유지. 정식 명세는 [[3. 프로젝트/spx-agent/references/audit-details-spec.md]] 참조.

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/data-mart.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/data-mart.md|Tasks]]
> 상위 본문: [[3. 프로젝트/spx-agent/hdd/design.md]] § 2.5 마트 설계
> 결정 노트: [[0. Inbox/마트 설계 결정 - 2026-05-14.md]]

> ⚠️ 5/8 4-fact / 5/11 OLTP 3-fact / 5/12 옵션 D 전부 폐기. **audit 단일 SoT + RBAC JOIN** (2026-05-12 이사님 확정).
> 본 문서는 5/14 옵션 B (사전 JOIN하되 ID만) + 2계층 마트 + cron chain refresh + **public schema 통일 + `spx_` prefix** + 5/15 결정 4건 적용 완료본.

## 0. 명명 규칙 (2026-05-14)

- 모든 객체 `public` schema (별도 audit/mart schema 폐기)
- 우리 객체에 `spx_` prefix — Dify upstream / 회사 RBAC 표준과 시각적 구분
- 매핑: `audit_events` → **`spx_audit_events`** / `v_audit_enriched` → **`spx_mv_audit_enriched`** / `v_resource_ownership_enriched` → **`spx_v_resource_ownership_enriched`** / `mv_kpi_calls_daily` → **`spx_mv_kpi_calls_daily`** / `mv_model_tokens_daily` → **`spx_mv_model_tokens_daily`** / refresh 트리거 = collector worker (pg_cron 미채택, 2026-05-15 결정)
- `public.` prefix는 PostgreSQL search_path 기본값이라 통상 생략 가능, DDL 작성 시 명시는 선택

## 1. 아키텍처 개요 — 2계층 마트

```
[원본]    public.spx_audit_events (Generated Column 8개 보강)
           + public.{resource_ownership, departments, department_members} (회사 RBAC)
              │
              ▼
[Layer 1] public.spx_mv_audit_enriched               ← MView, 이벤트 1건 = 1행 (event-level)
[Layer 1] public.spx_v_resource_ownership_enriched  ← View, 자산 ↔ 부서 매핑 (state)
              │
              ▼
[Layer 2] public.spx_mv_kpi_calls_daily             ← MView, 호출 메트릭 공용 큐브
[Layer 2] public.spx_mv_model_tokens_daily          ← MView, 모델 코스트 전용 큐브
              │
              ▼
            Backend Service (dashboard_*_service.py — Layer 2만 enforce)
              │
              ▼
            Frontend (TanStack Query staleTime 5min)
```

### 1.1 객체별 비즈니스 의미

| 객체 | 형태 | 한 줄 의미 |
|---|---|---|
| `spx_audit_events` Generated Column 8개 | 컬럼 | details JSONB 속 분기/필터 키를 인덱스 가능 컬럼으로 노출해 H-DASH-01/03 방어를 일관 적용 |
| `public.spx_mv_audit_enriched` | MView | 감사 이벤트 1건 = 1행. actor·owner 부서 ID, `is_debug`/`is_canonical_call` 플래그 사전 결합된 *정규화 이벤트 스트림* — 모든 호출 메트릭의 단일 진실원 (부서명은 query-time JOIN) |
| `public.spx_v_resource_ownership_enriched` | View | 현재 시점의 자산(앱/KB/도구) ↔ 소유 부서 매핑 *state 스냅샷* |
| `public.spx_mv_kpi_calls_daily` | MView | (일 × actor부서ID × owner부서ID × 앱 × app_mode) 단위 호출/사용자/에러/토큰 roll-up 큐브 |
| `public.spx_mv_model_tokens_daily` | MView | (일 × 모델제공자 × 모델ID) 단위 토큰/호출 합계 — message_send만 |

### 1.2 왜 2계층인가

- Layer 1만으로 5개 컴포넌트 다 쓸 수 있지만, kpi-cards + drill-through는 동시 조회라 같은 집계를 카드별로 4번 깎으면 비쌈
- Layer 2(daily roll-up mat)로 미리 day 단위까지 굴려두면 카드·차트·표 모두 같은 mat을 다른 GROUP BY로 재사용
- 역할 분리: Layer 1 = "정규화된 이벤트 스트림", Layer 2 = "비즈니스 큐브"
- 미래 진화 경로(incremental fact / 분석 DB)에서 Layer 2만 갈아끼우면 컴포넌트 코드 무수정

### 1.3 MView vs View 선택 기준 (3축)

| 축 | View 유리 | MView 유리 |
|---|---|---|
| 원본 크기 | 작다 (수천 이하) | 크다 (수십만 이상) |
| 신선도 허용치 | 즉시 필요 | 분 단위 OK |
| 조회 빈도 | 낮음 | 높음 |

- `audit_enriched`: 9백만 행 + 5분 stale OK + 5개 컴포넌트가 다 씀 → **MView**
- `ownership_enriched`: 수천 행 + RBAC 변경 즉시 반영이 자연스러움 → **View**

## 2. DDL

### 2.1 spx_audit_events Generated Column 8개 (2026-05-15 적용 완료)

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

> 컨벤션: details 추출 컬럼은 `_d` 접미사 (8개 중 7개). `target_app_id`만 예외 — top-level `target_id` + `details.appId` 혼용. 사유 + collector 보강 의존 매트릭스: `references/audit-details-spec.md § Generated Column 8개 DDL`

### 2.2 v_audit_enriched (Layer 1, MView, 옵션 B — ID만)

> Generated Column `_d` 접미사를 enriched view 안에서 마트 친화 alias(`app_mode`/`total_tokens` 등)로 노출 — 컴포넌트 쿼리 단순화.

```sql
CREATE MATERIALIZED VIEW public.spx_mv_audit_enriched AS
SELECT
  ae.id, ae.occurred_at, ae.tenant_id,
  ae.action, ae.actor_type, ae.actor_id,
  ae.target_app_id,                                  -- UUID, top-level + details 혼용 (접미사 없음)
  ae.app_mode_d         AS app_mode,                 -- alias 권장
  ae.model_provider_d   AS model_provider,
  ae.model_id_d         AS model_id,
  ae.total_tokens_d     AS total_tokens,             -- BIGINT
  ae.invoke_from_d      AS invoke_from,
  ae.triggered_from_d   AS triggered_from,
  ae.error_d            AS error_text,
  ro.owner_department_id AS app_owner_dept_id,      -- ID만 (이름은 query-time JOIN)
  dm.department_id       AS actor_dept_id,           -- ID만
  (ae.invoke_from_d = 'debugger'
    OR ae.triggered_from_d IN ('debugging','rag-pipeline-debugging')) AS is_debug,
  (ae.app_mode_d <> 'advanced-chat' OR ae.action='message_send') AS is_canonical_call
FROM public.spx_audit_events ae
JOIN public.resource_ownership ro                 -- INNER (결정 a)
       ON ro.tenant_id=ae.tenant_id AND ro.resource_type='app' AND ro.resource_id=ae.target_app_id
LEFT JOIN public.department_members dm            -- LEFT (end_user는 부서 없을 수 있음)
       ON dm.tenant_id=ae.tenant_id AND dm.account_id=ae.actor_id AND dm.is_active=TRUE
WHERE ae.action IN ('message_send','workflow_execute');   -- 결정 c

CREATE UNIQUE INDEX ON public.spx_mv_audit_enriched (id);  -- CONCURRENTLY refresh 전제
```

> **`is_debug` 표현식 (5/15 결정)**: `WorkflowRunTriggeredFrom` enum 7종 중 디버깅 의미 2종 모두 포함 (`debugging` + `rag-pipeline-debugging`). 5/14 코드 기반 재조사 발견.

**JOIN 의미**:
- `resource_ownership` INNER = "매칭되어야 정상" (결정 a) → 매칭 안 되면 모니터링 #3 캐치
- `department_members` LEFT = "end_user/api/system은 매칭 안 되는 게 자연" → 외부 액터 호출도 owner 기준 집계에서 누락 안 됨

**플래그 컬럼화 사유**: 컴포넌트가 N명 × N시간 추가됨. 마트 인터페이스에 흡수 안 하면 3개월 후 누군가 H-DASH-01 모르고 SQL 추가하며 이중카운트. "비즈니스 규칙은 마트 안에 흡수하고 컴포넌트는 SELECT만 한다"가 황금 규칙.

### 2.3 mv_kpi_calls_daily (Layer 2, 옵션 B)

```sql
CREATE MATERIALIZED VIEW public.spx_mv_kpi_calls_daily AS
SELECT
  tenant_id,
  date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul') AS day,
  app_owner_dept_id, actor_dept_id,
  target_app_id, app_mode,
  COUNT(*) AS calls,
  COUNT(DISTINCT actor_id) AS users,
  COUNT(*) FILTER (WHERE error_text IS NOT NULL) AS errors,
  SUM(total_tokens) AS tokens,
  MAX(occurred_at) AS last_seen
FROM public.spx_mv_audit_enriched
WHERE NOT is_debug AND is_canonical_call
GROUP BY 1,2,3,4,5,6;

CREATE UNIQUE INDEX ON public.spx_mv_kpi_calls_daily
  (tenant_id, day, app_owner_dept_id, actor_dept_id, target_app_id, app_mode);
```

타임존 = `Asia/Seoul` day 경계 (사용자 인지와 일치). 카디널리티: 부서 N개면 actor × owner = N². N=100 + day 30 + 앱 수십 + app_mode 3 = 백만 행 이내.

**컴포넌트 쿼리 예 (부서명 query-time JOIN)**:

```sql
SELECT d.name AS dept, SUM(c.calls) AS calls, SUM(c.tokens) AS tokens
FROM public.spx_mv_kpi_calls_daily c
JOIN public.departments d ON d.id = c.app_owner_dept_id   -- 매번 JOIN, departments 작아서 무비용
WHERE c.tenant_id=:tid AND c.day BETWEEN :start AND :end
GROUP BY d.name;
```

### 2.4 mv_model_tokens_daily (Layer 2)

```sql
CREATE MATERIALIZED VIEW public.spx_mv_model_tokens_daily AS
SELECT
  tenant_id,
  date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul') AS day,
  COALESCE(model_provider,'미분류') AS model_provider,
  COALESCE(model_id,'미분류')       AS model_id,
  SUM(total_tokens) AS tokens,
  COUNT(*)          AS calls
FROM public.spx_mv_audit_enriched
WHERE NOT is_debug AND action='message_send' AND total_tokens IS NOT NULL
GROUP BY 1,2,3,4;

CREATE UNIQUE INDEX ON public.spx_mv_model_tokens_daily
  (tenant_id, day, model_provider, model_id);
```

`kpi_calls_daily`와 별도 mat 사유: 그룹핑 차원 완전 상이 (calls=부서×앱 / tokens=모델). 합치면 카디널리티 폭증. workflow_execute는 모델 정보 구조적 미가용 → message_send만 (H-DASH-02).

### 2.5 v_resource_ownership_enriched (Layer 1, View)

```sql
CREATE VIEW public.spx_v_resource_ownership_enriched AS
SELECT
  ro.tenant_id, ro.resource_type, ro.resource_id,
  ro.owner_department_id, d.name AS owner_dept_name,
  ro.owner_account_id, ro.created_at
FROM public.resource_ownership ro
JOIN public.departments d ON d.id=ro.owner_department_id;
```

수백~수천 행 규모. View로도 풀스캔 비용 무시. RBAC 변경 즉시 반영.

## 3. 인덱스 정책

### 3.1 spx_audit_events 인덱스 (2026-05-15 적용 완료)

원본 테이블 가속용 5개:

| 인덱스 | 정의 | 용도 |
|---|---|---|
| `spx_audit_events_action_mart_idx` | partial `(action) WHERE action IN ('message_send','workflow_execute')` | enriched 메인 필터 |
| `spx_audit_events_tenant_target_app_idx` | `(tenant_id, target_app_id)` | resource_ownership JOIN |
| `spx_audit_events_tenant_actor_idx` | `(tenant_id, actor_id)` | department_members JOIN |
| `spx_audit_events_occurred_action_idx` | `(occurred_at, action)` | 미래 incremental refresh |
| `spx_audit_events_app_mode_idx` | `(app_mode_d)` | `is_canonical_call` 분기 |

### 3.2 Layer 1/2 mat 인덱스 (생성 예정)

| 대상 | 인덱스 | 용도 |
|---|---|---|
| `public.spx_mv_audit_enriched` | UNIQUE (id) | CONCURRENTLY refresh 전제 |
| `public.spx_mv_audit_enriched` | (tenant_id, occurred_at) | drill 정확 users 카운트 |
| `public.spx_mv_kpi_calls_daily` | UNIQUE (tenant_id, day, app_owner_dept_id, actor_dept_id, target_app_id, app_mode) | CONCURRENTLY + 컴포넌트 쿼리 |
| `public.spx_mv_kpi_calls_daily` | (tenant_id, day) | 전체 합계 |
| `public.spx_mv_model_tokens_daily` | UNIQUE (tenant_id, day, model_provider, model_id) | CONCURRENTLY |

> 추가 부분 인덱스(예: `WHERE NOT is_debug`)는 Layer 2 부하 테스트 후 확정.

## 4. ETL = Collector trigger 연동 (5분 polling cycle 끝에 chain refresh)

> 2026-05-15 결정 변경. pg_cron 미채택. dify-audit db-poller가 polling cycle 마지막에 직접 마트 chain refresh 호출.

### 4.1 단일 chain — collector worker 안에서 호출

```typescript
// dify-audit/src/workers/db-poller.ts (개념 윤곽)
async function refreshMartChain() {
  try {
    await prisma.$executeRawUnsafe(
      `REFRESH MATERIALIZED VIEW CONCURRENTLY public.spx_mv_audit_enriched`
    );
    await prisma.$executeRawUnsafe(
      `REFRESH MATERIALIZED VIEW CONCURRENTLY public.spx_mv_kpi_calls_daily`
    );
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

**왜 chain인가**: mat 의존 순서(enriched → daily mat) 보장. 분리 호출(병렬)이면 daily mat이 enriched 이전 스냅샷 기반으로 돌 위험. 순차 호출로 같은 enriched 스냅샷 보장.

### 4.2 왜 collector trigger인가 (pg_cron 미채택 사유)

- 우리 환경 `postgres:15-alpine`은 pg_cron 미포함 → image 교체 또는 Dockerfile build 필요. 인프라 부담 추가
- collector polling이 5분 주기인데 마트 cron이 별도로 도는 의미가 약함 — collector worker 죽으면 신규 데이터도 안 들어오고 마트 refresh도 의미 없음
- 코드 5줄로 끝, design.md 한 섹션 갱신만으로 적용 가능
- 평균 lag 1~2분 짧음 (collector run 끝 즉시 refresh, 별도 cron tick 대기 X)
- 운영 진입 직전 책임 분리 + 모니터링 메트릭 #1a/#1b 분리 가치 커지면 pg_cron 마이그레이션 검토 (한 방향 길이 막히지 않음)

### 4.3 왜 cron 배치 (실시간 미채택)

- 전체 lag = MAX(병목 단계). collector 5분 폴링이 진짜 병목이라 마트만 실시간 만들어도 사용자 체감 0
- TanStack staleTime 5분과 정합
- 진화 경로 (incremental refresh / pg_trigger / 분석DB)는 § 7 참조

### 4.4 CONCURRENTLY 다운타임 0

- 모든 MView에 UNIQUE INDEX 필수
- refresh 중에도 SELECT 가능
- 전제: UNIQUE INDEX 미설치 시 `non-CONCURRENTLY` fallback → SELECT 블록

### 4.5 에러 처리 정책

- 마트 refresh 실패가 collector polling cycle 다음 회차를 차단하면 안 됨 (try/catch + 로그만)
- 연속 N회 실패 시 알림 (별도 트랙)
- DB 연결 끊김 / 타임아웃 / lock 충돌 → collector 재시작 시 자연 회복 (다음 polling cycle에서 재시도)

## 5. 모니터링 메트릭 6개

> lag 단계 분리로 진단 효율 확보 (어디서 막혔는지 즉시 식별).

| # | 메트릭 | 임계 | 사고 종류 |
|---|---|---|---|
| 1a | **collector lag**: `now() - max(occurred_at) FROM spx_audit_events` | > 10분 | 폴링 5분 + buffer 5분 |
| 1b | **mart lag**: `max(occurred_at FROM spx_audit_events) - max(occurred_at FROM spx_mv_kpi_calls_daily)` | > 7분 | chain refresh 5분 + buffer 2분 |
| 2 | chain REFRESH 소요 (collector 로그 `[mart] chain refresh completed` 직전 시각 - `[db-poller] Run completed` 시각 차) | > 4분 | 1분 여유 위반 |
| 3 | enriched vs src 행수 차이 (대상 action 한정) | > 0 | ownership 누락 사고 (적재 단계) |
| 4 | `resource_ownership.owner_department_id IS NULL` 카운트 | > 0 | 부서 누락 사고 (등록 단계) |
| 5 | enriched의 actor_dept_id IS NULL & actor_type='account' 비율 | > 5% | department_members 동기화 사고 |

### 3단계 사고 캐치 메타

- **#3 적재 단계**: 이벤트의 target_app_id가 RBAC ownership에 미등록 → INNER JOIN으로 enriched에서 사라진 행 감지
- **#4 등록 단계**: ownership은 등록됐는데 부서 컬럼 NULL → KPI #1 총 오브젝트 누락
- **#5 동기화 단계**: 자산은 OK인데 사용자 부서 매핑 누락 → KPI #2 채택 카드에서 분류 못 됨

→ 세 메트릭이 **마트 건강 검진 체계**. 마트 객체 자체는 사고 나도 묵묵히 돌아감 → 메트릭이 옆에서 watching해서 *마트가 거짓말하는 순간*을 감지.

### RBAC 변경 stale의 두 시나리오

| 시나리오 | 변경 대상 | DB 시점 stale | 사용자 체감 stale |
|---|---|---|---|
| 인사이동 (김대리 IT → MKT) | `department_members.department_id` | 최대 5분 | 최대 5분 |
| 부서명 변경 (IT → IT본부) | `departments.name` | **0초** | 최대 5분 (프론트 캐시만) |

⚠️ 인사이동 한계: refresh 시 김대리가 IT 시절에 한 과거 호출도 MKT로 재분류됨 (SCD type 2 미적용).

## 6. 백엔드 서비스 변경

### 6.1 변경 패턴 (Layer 2만 enforce)

```python
# Before: OLTP 직접 조회
class DashboardKpiService:
    def get_call_count(self, start, end):
        return db.session.query(Message).filter(...).count() + ...

# After: Layer 2 mart 조회 + 부서명 query-time JOIN
class DashboardKpiService:
    def get_call_count(self, start, end, owner_dept_filter=None):
        q = (db.session.query(func.sum(MvKpiCallsDaily.calls))
             .filter(MvKpiCallsDaily.day.between(start, end)))
        if owner_dept_filter:
            q = q.join(Department, Department.id == MvKpiCallsDaily.app_owner_dept_id)
            q = q.filter(Department.name == owner_dept_filter)
        return q.scalar() or 0
```

### 6.2 신선도 표기 (fallback 없음)

- 마트 stale = mart_lag 메트릭으로 운영자 알림
- 사용자 화면에는 "마지막 갱신: HH:MM" 표기 (collector 로그 `[mart] chain refresh completed` 시각 또는 별도 `spx_mart_refresh_log` 테이블 도입 검토)
- 마트 다운 시 OLTP fallback 미적용 (audit 단일 SoT 원칙 — OLTP 직조회는 H-DASH-01/03 방어 누락 위험)

## 7. 실시간 확장 경로

```
[현재]  Collector trigger 연동 5분 (db-poller worker 안에서 chain refresh)
   ↓ 부하 증가 / lag 5분 부족
[A]    각 mat에 incremental refresh (last_refreshed_at 컬럼)
   ↓ 실시간성 요구 (30초 이내 반영)
[B]    PG logical replication / pg_trigger → Redis Stream
            → 별도 fact 테이블 incremental upsert
   ↓ 글로벌 / 멀티테넌트 스케일
[C]    분석 DB 분리 (ClickHouse / TimescaleDB)
```

핵심: Layer 2가 컴포넌트의 인터페이스 — `spx_mv_kpi_calls_daily`만 보면 아래 구현이 MView → incremental fact → 분석DB로 바뀌어도 컴포넌트 코드 무수정.

## 8. 마이그레이션

> 도구: **Prisma Migrate** (Alembic 아님). 마이그레이션 폴더 = `dify-audit/prisma/audit/migrations/`. Generated Column은 Prisma ORM 직접 미지원 → raw SQL migration 패턴. 상세: `references/audit-details-spec.md § 마이그레이션 절차 (Prisma Migrate)`.

### 8.1 완료 단계 (2026-05-15)

- ✅ `audit` schema → `public` ren​ame + audit schema `DROP CASCADE`
- ✅ collector 4 patch (`messages.ts` / `workflow-runs.ts` / `workflow-nodes.ts` / `conversations.ts`)
- ✅ Generated Column 8개 (`_d` 접미사) + 인덱스 5개 ALTER
- ✅ trigger 함수 rename `audit.log_dify_change()` → `public.spx_log_dify_change()` + trigger 5종 재배포

### 8.2 잔여 단계

- ⏳ Layer 1 객체 (`spx_mv_audit_enriched` + `spx_v_resource_ownership_enriched`) + UNIQUE INDEX
- ⏳ Layer 2 mat 2종 + UNIQUE INDEX
- ⏳ Collector trigger 연동 chain refresh (db-poller 코드에 `refreshMartChain()` 호출 추가)
- ⏳ downgrade 작성 (DROP MATERIALIZED VIEW, DROP VIEW, cron.unschedule — Generated Column은 ALTER TABLE DROP COLUMN 역순)

### 8.3 운영 적용 시 주의 (5/15 사고 학습)

- `audit_writer` role은 ALTER 권한 미보유 → 마이그레이션 적용 시 두 옵션:
  - (a) postgres superuser로 직접 DDL + `prisma migrate resolve --applied`로 정합 처리 (5/15 채택)
  - (b) `audit_writer`에게 임시 ALTER 권한 GRANT 후 회수
- `CREATE INDEX CONCURRENTLY`는 트랜잭션 밖이라 Prisma migration에서 사용 불가 → 운영 적용 시 별도 SQL로 수동 실행 권장

## 9. Harness 반영

requirements § "Harness 방어" 표 그대로. 마트 인터페이스에 흡수되는 형태:

- H-DASH-01 → enriched `is_canonical_call` 컬럼
- H-DASH-03 → enriched `is_debug` 컬럼 + daily mat `WHERE NOT is_debug`
- H-DASH-04 → INNER JOIN ro (결정 a, NULL은 모니터링 #3로 발견)
- H-DASH-02 → mv_model_tokens_daily는 `action='message_send'`만
- H-DASH-05 → 서비스 레이어에서 N/A 분기 (마트 외부)

코드 리뷰 시 grep 검증:
```bash
grep -E "details->>'(invokeFrom|triggeredFrom|appMode)'" api/services/admin/    # 컴포넌트가 raw 추출하면 위반
grep -E "(app_mode_d|triggered_from_d|invoke_from_d)" api/services/admin/       # 마트 인터페이스 우회하면 위반 (enriched alias만 사용해야 함)
grep -E "spx_mv_audit_enriched|spx_mv_kpi_calls_daily" api/services/admin/       # Layer 2 사용 확인
```

## 10. 미해결 결정 — design.md § 10 참조

drill-through users 정확도 / Read Replica / 타임존 / dbt / PM 확인 2건 모두 design.md § 10 미해결 결정 표 참조.

## 참조

- requirements: [[3. 프로젝트/spx-agent/hdd/specs/requirements/data-mart.md]]
- tasks: [[3. 프로젝트/spx-agent/hdd/specs/tasks/data-mart.md]]
- 상위 본문: [[3. 프로젝트/spx-agent/hdd/design.md]] § 2.5
- 5/14 결정 노트: [[0. Inbox/마트 설계 결정 - 2026-05-14.md]]
- audit 필드 명세: `references/audit-details-spec.md`
- 인벤토리: [[3. 프로젝트/spx-agent/references/dashboard-query-inventory.md]]
- 회의록: [[2. 회의록/0507 AAI 주간 보고]]
