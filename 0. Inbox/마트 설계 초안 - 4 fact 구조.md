---
tags: [프로젝트, dify, AI-Agent, HDD, 임시]
date: 2026-05-08
last_updated: 2026-05-11
status: 설계만 design.md § 2로 이관. **구현은 보류** (5/11 2단계 측정 결과 반영)
related:
  - "[[3. 프로젝트/spx-agent/hdd/specs/design/data-mart.md]]"
  - "[[0. Inbox/대시보드 12종 OLTP vs audit 쿼리 비교]]"
  - "[[0. Inbox/회의 답변 초안 - 데이터 마트 1단계 결과.md]]"
  - "[[4. 지식노트/Dify - 데이터 소스 3종 비교 (OLTP vs 로그파일 vs 내부 로그 테이블)]]"
  - "[[4. 지식노트/Dify - 대시보드 API 백엔드 2.7초 병목 분석 (Flask-Login 캐시 미작동)]]"
---

# 마트 설계 초안 — 4 fact 구조 (OLTP 3 + audit 1) + RBAC dim

> ⚠️ **5/11 최종 갱신 — 설계 단순화 + audit 부분 보류**
>
> **확정**: OLTP 3-fact로 단순화 (4-fact에서 `fact_audit_event_daily` 제거)
>   - `fact_call_daily` / `fact_user_activity_daily` / `fact_objects_snapshot_daily` (Gold/Silver 메달 용어 안 씀)
>   - 차트 ①~⑱ 커버
>
> **보류**: top-error-types(차트 ⑲) + 드로어 화면의 audit 활용 방식
>   - 옵션 C(audit collector 보강) / D(OLTP 정규식 분류) / E(절충) 중 audit collector 실제 동작 확인 후 결정
>   - 확인: `SELECT DISTINCT action FROM audit.audit_events` (현재 본인 브랜치엔 audit 없음, dev 머지 후 재개)
>
> **구현 시점**: 운영 데이터 양 도달 시 (마트화 즉시 효과는 현 mock에서 0%, 단 장기 보험으로 미리 구현)
>
> **재측정 트리거**: messages 10만 건 도달 OR 백엔드 응답시간 중 DB 비중 30% 이상
>
> 상세: [[0. Inbox/회의 답변 초안 - 데이터 마트 1단계 결과.md]]

> **배경**: 2026-05-08 이사님 컨펌 — audit/OLTP/RBAC 섞어 마트 만들기. 단 audit의 5분 폴링·90일 보존은 로그 특성이라 자연스럽게 받아들이고, jsonb 추출 비용 때문에 jsonb 의존 차트는 OLTP 사용.

## 핵심 원칙

### 단일 통합 fact 금지
audit(90일) vs OLTP(영구) **보존 정책이 달라** 한 테이블로 못 묶음.
- 단일 fact에 합치면 audit 부분만 91일째 사라지면서 OLTP 부분만 남는 비대칭 → 카운트 부정확
- → **fact는 데이터 출처별로 분리**, 차트가 필요한 fact를 골라 쓰는 구조

### RBAC는 dim 역할 (별도 fact 없음)
`departments`, `department_members`, `resource_ownership`은 OLTP에 그대로 두고:
- ETL 시점에 fact 행에 `dept_id`를 미리 박음 → 화면 조회 시 JOIN 없이 GROUP BY 가능
- 부서 매핑이 시간에 따라 바뀌면 SCD(Slowly Changing Dimension) 정책 별도 결정 필요 (§ 미해결)

## 4-fact 구조

### 1. `fact_call_daily` — 호출/토큰 (OLTP 기반, 영구)

```sql
CREATE TABLE fact_call_daily (
    date           DATE         NOT NULL,
    dept_id        UUID,         -- NULL = 미배정 (H-DASH-04)
    app_id         UUID         NOT NULL,
    app_mode       VARCHAR(32)  NOT NULL,
    model_id       VARCHAR(64),  -- NULL = workflow 또는 미분류 (H-DASH-02)
    msg_count      INTEGER      NOT NULL DEFAULT 0,
    wf_count       INTEGER      NOT NULL DEFAULT 0,
    error_count    INTEGER      NOT NULL DEFAULT 0,
    token_sum      BIGINT       NOT NULL DEFAULT 0,
    cost_sum       NUMERIC(12,4) NOT NULL DEFAULT 0,
    PRIMARY KEY (date, dept_id, app_id, model_id)
);

CREATE INDEX idx_fact_call_dept_date ON fact_call_daily (dept_id, date DESC);
CREATE INDEX idx_fact_call_date ON fact_call_daily (date DESC);
CREATE INDEX idx_fact_call_model ON fact_call_daily (model_id, date DESC);
```

**ETL 소스**: `messages` + `workflow_runs` JOIN apps JOIN resource_ownership
- 디버거 필터 박음 (`invoke_from != 'debugger'`, `triggered_from = 'app-run'`)
- AppMode 분기 (H-DASH-01: ADVANCED_CHAT은 messages만 카운트)

**커버하는 차트**:
- KPI: API 호출, 에러율 (임의 기간 가능 — 24h 고정 풀림)
- model-tokens 차트
- model-call-share, dept-call-count, dept-call-rps, dept-error-rate, dept-error-table

### 2. `fact_user_activity_daily` — 활성 사용자 (OLTP 기반, 영구)

```sql
CREATE TABLE fact_user_activity_daily (
    date           DATE         NOT NULL,
    dept_id        UUID,
    user_id        UUID         NOT NULL,
    msg_count      INTEGER      NOT NULL DEFAULT 0,
    is_first_seen  BOOLEAN      NOT NULL DEFAULT FALSE,  -- 신규 판별
    PRIMARY KEY (date, dept_id, user_id)
);

CREATE INDEX idx_fact_user_dept_date ON fact_user_activity_daily (dept_id, date DESC);
```

**ETL 소스**: `messages.from_end_user_id` UNION `workflow_runs.created_by` GROUP BY (date, user_id)
- `from_end_user_id IS NOT NULL` (H-DASH-08)
- 신규 판별: `accounts.created_at` 기준으로 first_seen 결정

**커버하는 차트**:
- KPI: 활성 사용자
- dept-dau, dept-user-activity-table (DAU/WAU/신규/이탈)
- model-users (model 차원 추가하면)

> 메모: model-users는 user×model GROUP BY라 `fact_user_model_daily` 분리 검토 후보. 측정 후 결정.

### 3. `fact_objects_snapshot_daily` — 오브젝트 누적 (OLTP 기반, 영구)

```sql
CREATE TABLE fact_objects_snapshot_daily (
    date              DATE         NOT NULL,
    dept_id           UUID,
    resource_type     VARCHAR(32)  NOT NULL,  -- 'app'/'dataset'/'tool'
    cumulative_count  INTEGER      NOT NULL DEFAULT 0,
    new_count         INTEGER      NOT NULL DEFAULT 0,  -- 그 일자 신규 생성
    PRIMARY KEY (date, dept_id, resource_type)
);
```

**ETL 소스**: `apps` + `datasets` + `tools` + `resource_ownership`
- 매일 일자별 누적 스냅샷 (created_at <= 그 날까지)
- 신규는 `created_at = 그 날`

**커버하는 차트**:
- KPI: 총 오브젝트
- dept-cumulative, top-owners (top-owners는 user별이라 별도 view 검토)
- dept-new-creations-table

> 메모: top-owners는 사용자×resource_type GROUP BY가 본질이라 별도 `fact_owner_snapshot_daily` 후보. 측정 후 결정.

### 4. `fact_audit_event_daily` — 감사 이벤트 (audit 기반, 90일 자연 cap)

```sql
CREATE TABLE fact_audit_event_daily (
    date           DATE         NOT NULL,
    dept_id        UUID,         -- actor → department_members 매핑
    action         VARCHAR(64)  NOT NULL,  -- 'auth_failed', 'rate_limit_exceeded', 'prompt_update' 등
    category       VARCHAR(32)  NOT NULL,  -- 'admin'/'user'/'security'
    event_count    INTEGER      NOT NULL DEFAULT 0,
    failed_count   INTEGER      NOT NULL DEFAULT 0,
    PRIMARY KEY (date, dept_id, action)
);

CREATE INDEX idx_fact_audit_action ON fact_audit_event_daily (action, date DESC);
```

**ETL 소스**: `audit.audit_events`
- 보존: audit 자체가 90일 → 이 fact도 자연스럽게 90일 max
- 화면 기간 필터 max 90일로 제한

**커버하는 차트**:
- top-error-types (현 12종 중 유일한 audit 차트)
- **향후 추가 차트 후보** (현 12종 외):
  - 인증 실패 / Rate limit 추이
  - 거버넌스 이력 (프롬프트 수정, 앱 생성·삭제, API 토큰 발급)
  - 멤버 권한 변경 이력

## ETL 배치

### 주기 — 1시간 (초안)
- Celery beat: 매시 0분
- 측정 후 단축 검토 (5분 / 15분)

### 증분 vs 전체
- **증분** 기본: 어제 + 오늘만 재계산 (오늘 분은 매시간 갱신, 어제 분은 새벽 1회 백필)
- 백필 명령: `flask mart backfill --fact <name> --start <date> --end <date>`

### Idempotent
모든 fact에 `INSERT ... ON CONFLICT (PK) DO UPDATE` 패턴.

### 신선도 추적
`mart_refresh_log` 테이블 — fact_name별 last_success / status / rows_affected 기록.

## 화면 표기 (이사님 2번 결정 반영)

### 기간 필터 — fact별 max 다름
- OLTP 기반 3 fact: max 1년 (또는 무제한)
- audit 기반 fact_audit_event_daily: **max 90일** (자연 cap, UI에서 제한)
- 에러율 KPI: **24h 고정 → 기간 필터 가능하게 변경** (fact_call_daily의 error_count로 임의 기간 계산)

### 신선도 표시
대시보드 헤더 우측에 "마지막 갱신: HH:MM" — fact 중 가장 오래된 것 기준.

## Fallback 정책

§ 6.2 옵션 B 유지 (신선도 < 1h 마트, 아니면 OLTP 직접). 단:
- audit 기반 fact는 **fallback 없음** (OLTP에 audit 정보가 없으니) → audit fact 다운 시 그 차트만 "데이터 없음" 표시

## RBAC dept_id 매핑 정책 (이사님 4번 결정 반영)

ETL 시점에 fact 행에 `dept_id` 미리 박는 방식. 단:

### 어느 시점의 부서 매핑을 쓸 것인가?
1. **현재 소속**: ETL 실행 시점의 `department_members` 사용 (단순)
2. **이력 기반**: `department_members`의 변경 이력을 따라 그 일자의 매핑 사용 (정확하나 SCD 부담)

→ **초안: 현재 소속**. 일자 변환 로직 부담 줄이기 위해. 부서 이동 빈번하면 재검토.

### 미배정 처리 (H-DASH-04)
- `dept_id IS NULL`로 박음
- 화면에서 COALESCE(dept_name, '미배정')으로 fallback

## Harness 반영 (ETL SQL에 박기)

| ID | 방어 | ETL SQL 박는 곳 |
|----|------|----------------|
| H-DASH-01 | AppMode 분기 (ADVANCED_CHAT은 messages만) | `fact_call_daily` ETL의 wf_count 계산 시 |
| H-DASH-02 | model_id NULL → "미분류" | `fact_call_daily` ETL의 model_id 컬럼 |
| H-DASH-03 | 디버거 필터 | `fact_call_daily` ETL의 WHERE 절 |
| H-DASH-04 | 미배정 fallback | `fact_*` ETL의 dept_id NULL 허용 |
| H-DASH-08 | NULL 사용자 카운트 안 함 | `fact_user_activity_daily` ETL의 WHERE 절 |

## 미해결 결정 (측정 후 확정)

- [ ] `fact_user_model_daily` 분리 여부 (model-users 차트용)
- [ ] `fact_owner_snapshot_daily` 분리 여부 (top-owners 차트용)
- [ ] ETL 주기 1h vs 15min vs 5min
- [ ] dept_id 매핑 정책 — 현재 소속 vs 이력 기반 (SCD)
- [ ] audit 90일 max를 화면에서 어떻게 안내할지 (조용히 잘림 vs 명시 안내)
- [ ] 첫 마트 시범 — `fact_call_daily` 1개부터? 아니면 4개 동시 PoC?

## 회의 답변 ②③ 초안 (이 설계 기반)

### ② "마트 필요한 부분 + 유형별 스키마"
> 마트는 **4개 분리 — OLTP 기반 3개 + audit 기반 1개**.
> - 호출/토큰: `fact_call_daily` (시계열 집계형)
> - 활성 사용자: `fact_user_activity_daily` (사용자×일자 입도)
> - 오브젝트 누적: `fact_objects_snapshot_daily` (스냅샷형)
> - 감사 이벤트: `fact_audit_event_daily` (90일 cap, 거버넌스/보안용)
> RBAC(부서)는 별도 fact 없이 ETL 시점에 fact 행에 dept_id 박음.

### ③ "마트로 12종 다 그릴 수 있나?"
> **그릴 수 있음**. 12종 모두 4 fact 안에서 커버.
> 단:
> - 응답 시간 < 3s 목표는 마트만으로 부족 → **인덱스 + Redis 캐시 + 마트 3축** 조합 필요
> - top-owners / model-users는 차원 GROUP BY가 추가라 별도 fact 분리 검토 (측정 후 결정)
> - 향후 거버넌스/보안 추가 차트는 fact_audit_event_daily 활용 가능
