---
tags: [프로젝트, dify, AI-Agent, HDD, 임시]
date: 2026-05-08
status: 검토 후 design.md 또는 references로 이관
related:
  - "[[3. 프로젝트/spx-agent/hdd/references/dashboard-query-inventory.md]]"
  - "[[4. 지식노트/Dify - 데이터 소스 3종 비교 (OLTP vs 로그파일 vs 내부 로그 테이블)]]"
  - "[[3. 프로젝트/spx-agent/references/audit-schema.md]]"
---
# 대시보드 12종 — OLTP vs audit 쿼리 비교

> 목적: 회의 답변 ① "어느 테이블에서 가져올까" 근거 자료. 12종 각각에 대해 OLTP 직접 조회 vs audit_events 조회 쿼리를 나란히 박아 비교.
> 결론 미리: **11개 OLTP + 1개 audit(top-error-types) 권장**. 나머지 11개도 audit으로 그릴 순 있으나 jsonb 추출/JOIN 비용/누적 카운트 한계 때문에 OLTP가 자연스러움.

## 비교 기준

audit_events 컬럼 (참고용):
- `action` text — 분류 (`message_send`, `workflow_execute`, `app_create`, `auth_failed` 등)
- `actor_id` text — Keycloak user_id 또는 end_user_id
- `target_type` / `target_id` text — 대상 (app/dataset/endpoint)
- `status` text — `success` / `failed`
- `source` text — `dify_db`(5분 폴링) / `pg_trigger` / `nginx_log` / `dify_audit_app`
- `occurred_at` timestamp
- `tenant_id` text
- `details` jsonb — 액션별 가변 메타 (modelId, totalTokens, error, invokeFrom 등)

`messages` collector의 `details` 페이로드 (확정):
```js
{
  messageId, conversationId,
  modelProvider, modelId,
  messageTokens, answerTokens, totalTokens,
  responseLatency, totalPrice, currency,
  error, fromSource, invokeFrom, workflowRunId
}
```

→ OLTP `messages` 테이블의 거의 모든 정보가 jsonb로도 보존됨.

`workflow-runs` collector의 `details` 페이로드 (확정):
```js
{
  workflowRunId,
  workflowId,
  elapsedTime,
  totalTokens,
  error,
}
```

→ **OLTP보다 정보 빈약**. 누락된 핵심 필드:
- ❌ `triggered_from` — **H-DASH-03 디버거 필터 핵심**. audit으론 디버거 호출 분리 불가
- ❌ `invoke_from` — 동일
- ❌ `total_price` / `currency` — 비용 계산 불가
- (model_id는 OLTP `workflow_runs`에도 원래 없음 — H-DASH-02 정상)

→ **결정적**: workflow 계열 차트(call-count / call-rps / error-rate / error-table)는 audit으로 가면 디버거 데이터 섞여 카운트 부정확.

## 1. objects 메트릭 (총 오브젝트)

### 1.1 dept-cumulative (objects 좌) — 부서별 누적 오브젝트

**의미**: 부서별로 보유한 app/dataset/tool 개수 누적

```sql
-- OLTP
SELECT COALESCE(d.name, '미배정') AS dept, COUNT(*) AS cnt
FROM resource_ownership ro
LEFT JOIN departments d ON ro.owner_department_id = d.id
GROUP BY d.name;

-- audit
SELECT
  COALESCE(d.name, '미배정') AS dept,
  SUM(CASE WHEN action LIKE '%_create' THEN 1 WHEN action LIKE '%_delete' THEN -1 END) AS cnt
FROM audit_events ae
LEFT JOIN departments d ON ...  -- actor → 부서 매핑 별도 필요
WHERE action IN ('app_create','app_delete','dataset_create','dataset_delete', ...)
GROUP BY d.name;
```

**차이**: ❌ **audit 부적절**. 90일 보존이라 91일 전 created entity가 빠지면 카운트 부정확. **누적 = (create−delete) 모든 시간**이라 보존 정책과 충돌.

---

### 1.2 top-owners (objects 우) — Top 10 소유자

**의미**: 앱·데이터셋·도구 가장 많이 만든 사람 Top 10

```sql
-- OLTP
SELECT a.name, d.name AS dept, COUNT(*) AS cnt
FROM resource_ownership ro
JOIN accounts a ON ro.owner_account_id = a.id
LEFT JOIN department_members dm ON dm.account_id = a.id
LEFT JOIN departments d ON dm.department_id = d.id
GROUP BY a.name, d.name
ORDER BY cnt DESC LIMIT 10;

-- audit
SELECT actor_email, COUNT(*) AS cnt
FROM audit_events
WHERE action IN ('app_create','dataset_create')
  AND status = 'success'
GROUP BY actor_email
ORDER BY cnt DESC LIMIT 10;
```

**차이**: ⚠️ audit 가능하나 **90일 보존 한계**. 91일 전 만든 앱 카운트 빠짐. 부서 라벨도 actor → 부서 매핑 별도 JOIN 필요.

---

### 1.3 dept-new-creations-table (objects 표) — 부서별 신규 생성

**의미**: 기간 내 새로 만든 app/dataset/tool 카운트

```sql
-- OLTP
SELECT COALESCE(d.name, '미배정') AS dept, ro.resource_type, COUNT(*) AS cnt
FROM resource_ownership ro
LEFT JOIN departments d ON ro.owner_department_id = d.id
WHERE ro.created_at BETWEEN $start AND $end
GROUP BY d.name, ro.resource_type;

-- audit
SELECT actor_email, target_type, COUNT(*) AS cnt
FROM audit_events
WHERE action IN ('app_create','dataset_create')
  AND occurred_at BETWEEN $start AND $end
GROUP BY actor_email, target_type;
```

**차이**: ✅ **둘 다 가능**. audit은 부서 매핑 별도. 기간 내라면 90일 한계 영향 없음.

---

## 2. users 메트릭 (활성 사용자)

### 2.1 dept-dau (users 좌) — 부서별 DAU

**의미**: 24h 내 활성 사용자 수 (부서별 DISTINCT)

```sql
-- OLTP
SELECT COALESCE(d.name, '미배정') AS dept,
       COUNT(DISTINCT m.from_end_user_id) AS dau
FROM messages m
LEFT JOIN department_members dm ON dm.account_id = m.from_account_id
LEFT JOIN departments d ON dm.department_id = d.id
WHERE m.invoke_from != 'debugger'
  AND m.from_end_user_id IS NOT NULL
  AND m.created_at > NOW() - INTERVAL '24 hours'
GROUP BY d.name;

-- audit
SELECT COALESCE(d.name, '미배정') AS dept,
       COUNT(DISTINCT actor_id) AS dau
FROM audit_events ae
LEFT JOIN department_members dm ON dm.account_id::text = ae.actor_id
LEFT JOIN departments d ON dm.department_id = d.id
WHERE action = 'message_send'
  AND details->>'invokeFrom' != 'debugger'
  AND occurred_at > NOW() - INTERVAL '24 hours'
GROUP BY d.name;
```

**차이**: ✅ 가능. ⚠️ audit는 `details->>'invokeFrom'` jsonb 추출 + `actor_id` text↔uuid 형변환 비용. 인덱스 안 박혀있으면 GROUP BY 풀스캔.

---

### 2.2 model-users (users 우) — 모델별 사용자 수

**의미**: 모델별 distinct 사용자

```sql
-- OLTP
SELECT m.model_id, COUNT(DISTINCT m.from_end_user_id) AS users
FROM messages m
WHERE m.invoke_from != 'debugger'
  AND m.from_end_user_id IS NOT NULL
GROUP BY m.model_id;

-- audit
SELECT details->>'modelId' AS model_id,
       COUNT(DISTINCT actor_id) AS users
FROM audit_events
WHERE action = 'message_send'
  AND details->>'invokeFrom' != 'debugger'
GROUP BY details->>'modelId';
```

**차이**: ✅ 가능. ⚠️ audit는 jsonb 표현식 GROUP BY → **표현식 인덱스 없으면 OLTP 대비 2~5배 느림**.

---

### 2.3 dept-user-activity-table (users 표) — DAU/WAU/신규/이탈

**의미**: 부서별 DAU(24h)/WAU(7d)/신규/이탈

```sql
-- OLTP (요약)
WITH period_users AS (
  SELECT m.from_end_user_id, m.created_at,
         dm.department_id
  FROM messages m
  LEFT JOIN department_members dm ON ...
  WHERE m.invoke_from != 'debugger' AND m.from_end_user_id IS NOT NULL
)
SELECT dept,
       COUNT DISTINCT user (24h) AS dau,
       COUNT DISTINCT user (7d) AS wau,
       NEW (NOT EXISTS prev period) AS new_users,
       CHURNED (prev YES, current NO) AS churned
FROM period_users
GROUP BY dept;

-- audit (구조 동일, messages → audit_events, 컬럼 jsonb 추출만 다름)
WITH period_users AS (
  SELECT actor_id, occurred_at, dm.department_id
  FROM audit_events
  LEFT JOIN department_members dm ON dm.account_id::text = ae.actor_id
  WHERE action = 'message_send'
    AND details->>'invokeFrom' != 'debugger'
)
-- 이하 동일
```

**차이**: ✅ 가능. ⚠️ **신규 판별 위험**: 90일 보존이라 90일 전 활성 user를 "신규"로 오판. OLTP는 `accounts.created_at`로 정확 판별.

---

## 3. calls 메트릭 (API 호출)

### 3.1 dept-call-count (calls 좌) — 부서별 호출수

**의미**: 부서별 API 호출 카운트 (AppMode 분기)

```sql
-- OLTP (3-CTE 요약)
WITH dept_apps AS (...),
     msg_calls AS (
       SELECT da.department_id, COUNT(*) AS cnt
       FROM messages m JOIN dept_apps da ON m.app_id = da.app_id
       WHERE m.invoke_from != 'debugger'
       GROUP BY da.department_id),
     wf_calls AS (
       SELECT da.department_id, COUNT(*) AS cnt
       FROM workflow_runs w
       JOIN apps a ON w.app_id = a.id AND a.mode = 'workflow'
       JOIN dept_apps da ON w.app_id = da.app_id
       WHERE w.triggered_from = 'app-run'
       GROUP BY da.department_id)
SELECT dept, COALESCE(msg.cnt,0) + COALESCE(wf.cnt,0) AS total
FROM dept_apps FULL OUTER JOIN ...;

-- audit
SELECT COALESCE(d.name, '미배정') AS dept, COUNT(*) AS cnt
FROM audit_events ae
LEFT JOIN department_members dm ON dm.account_id::text = ae.actor_id
LEFT JOIN departments d ON dm.department_id = d.id
WHERE action IN ('message_send','workflow_execute')
  AND details->>'invokeFrom' != 'debugger'
  AND status = 'success'
GROUP BY d.name;
```

**차이**: ✅ 둘 다 가능. **audit이 의외로 단순** (AppMode 분기를 action으로 자연 분리). 다만 부서 매핑이 actor 기준이라 OLTP의 "app→부서" 매핑과 의미 살짝 다름.

---

### 3.2 model-call-share (calls 우) — 모델별 호출 점유율

**의미**: 모델별 호출수 GROUP BY

```sql
-- OLTP
SELECT COALESCE(model_id, '미분류') AS model, COUNT(*) AS cnt
FROM messages
WHERE invoke_from != 'debugger'
GROUP BY model_id;

-- audit
SELECT COALESCE(details->>'modelId', '미분류') AS model, COUNT(*) AS cnt
FROM audit_events
WHERE action = 'message_send'
  AND details->>'invokeFrom' != 'debugger'
GROUP BY details->>'modelId';
```

**차이**: ✅ 가능. ⚠️ audit jsonb 추출 비용. workflow는 양쪽 모두 model 정보 없음 → "미분류" 버킷 동일.

---

### 3.3 dept-call-rps-table (calls 표) — 부서별 RPS + Top앱 + 추세

**의미**: 부서별 호출수 / RPS(=COUNT/86400) / Top앱(부서 내 1위) / 추세(현재 vs 직전 기간)

```sql
-- OLTP (7-CTE 골격)
WITH dept_apps AS (
  -- 부서별 앱 매핑 (resource_ownership)
  SELECT ro.owner_department_id AS dept_id, ro.resource_id AS app_id
  FROM resource_ownership ro WHERE ro.resource_type = 'app'
),
curr_calls AS (
  -- 현재 기간: messages + workflow_runs UNION ALL
  SELECT da.dept_id, da.app_id, m.created_at
  FROM messages m JOIN dept_apps da ON m.app_id = da.app_id
  WHERE m.invoke_from != 'debugger'
    AND m.created_at BETWEEN $start AND $end
  UNION ALL
  SELECT da.dept_id, da.app_id, w.created_at
  FROM workflow_runs w
  JOIN apps a ON w.app_id = a.id AND a.mode = 'workflow'
  JOIN dept_apps da ON w.app_id = da.app_id
  WHERE w.triggered_from = 'app-run'
    AND w.created_at BETWEEN $start AND $end
),
prev_calls AS (/* 같은 구조, 직전 기간 */),
curr_agg AS (
  SELECT dept_id, COUNT(*) AS curr_total FROM curr_calls GROUP BY dept_id
),
prev_agg AS (/* 동일 */),
dept_app_calls AS (
  SELECT dept_id, app_id, COUNT(*) AS app_calls,
         ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY COUNT(*) DESC) AS rn
  FROM curr_calls GROUP BY dept_id, app_id
),
top_apps AS (
  SELECT dept_id, app_id FROM dept_app_calls WHERE rn = 1
)
SELECT d.name AS dept,
       c.curr_total AS calls,
       (c.curr_total / 86400.0) AS rps,
       (SELECT a.name FROM apps a WHERE a.id = t.app_id) AS top_app,
       ((c.curr_total - p.prev_total) * 100.0 / NULLIF(p.prev_total, 0)) AS trend
FROM departments d
LEFT JOIN curr_agg c ON c.dept_id = d.id
LEFT JOIN prev_agg p ON p.dept_id = d.id
LEFT JOIN top_apps t ON t.dept_id = d.id;

-- audit (불가)
-- workflow audit엔 triggered_from이 details에 없어서
-- "WHERE details->>'triggeredFrom' = 'app-run'" 절 자체가 못 박힘
-- → 디버거 호출 섞인 채로 카운트 → RPS / 추세 부정확
```

**차이**: ❌ **audit 불가 (collector 보강 전)**. workflow `triggered_from` 누락이 분자 카운트를 오염시킴. + 7-CTE × jsonb/형변환 비용까지 누적.

---

## 4. errors 메트릭 (에러율 24h)

### 4.1 dept-error-rate (errors 좌) — 부서별 에러율

**의미**: 부서별 (실패 / 전체) × 100

```sql
-- OLTP
WITH dept_apps AS (...),
     msg_stats AS (
       SELECT da.department_id,
              COUNT(*) FILTER (WHERE m.error IS NOT NULL) AS errors,
              COUNT(*) AS total
       FROM messages m JOIN dept_apps da ...),
     wf_stats AS (
       SELECT da.department_id,
              COUNT(*) FILTER (WHERE w.status = 'failed') AS errors,
              COUNT(*) AS total
       FROM workflow_runs w ...)
SELECT dept, (errors / NULLIF(total,0)) * 100 AS rate
FROM ...;

-- audit
SELECT COALESCE(d.name, '미배정') AS dept,
       (COUNT(*) FILTER (WHERE status = 'failed') * 100.0 /
        NULLIF(COUNT(*), 0)) AS rate
FROM audit_events ae
LEFT JOIN department_members dm ON dm.account_id::text = ae.actor_id
LEFT JOIN departments d ON dm.department_id = d.id
WHERE action IN ('message_send','workflow_execute')
  AND occurred_at > NOW() - INTERVAL '24 hours'
GROUP BY d.name;
```

**차이**: ✅ 둘 다 가능. **audit이 더 단순** (status 컬럼 직접 활용, FILTER 절 단순). 부서 매핑 의미 차이 위와 동일.

---

### 4.2 top-error-types (errors 우) — 에러 유형 Top ⭐

**의미**: 가장 많은 에러 유형 분류

```sql
-- OLTP
SELECT m.error AS type, COUNT(*) AS cnt
FROM messages m
WHERE m.error IS NOT NULL
GROUP BY m.error
ORDER BY cnt DESC LIMIT 10;
-- 문제: m.error는 자유 텍스트 → 'timeout occurred at...' 같은 변동 메시지가 다 다른 type으로 묶임

-- audit ⭐
SELECT action AS type, COUNT(*) AS cnt
FROM audit_events
WHERE category = 'security'
   OR (status = 'failed' AND action != 'message_send')
GROUP BY action
ORDER BY cnt DESC LIMIT 10;
-- audit은 action이 이미 분류돼있음: auth_failed, rate_limit_exceeded, model_timeout 등
```

**차이**: 🌟 **audit 명확히 우월**. OLTP `messages.error`는 자유 텍스트라 분류 안 됨. audit `action`은 폴러가 이미 정규화. **이 1개만 audit 권장**.

---

### 4.3 dept-error-table (errors 표) — 부서별 에러+주원인+트렌드

**의미**: 부서별 실패수 / 에러율 / 주 원인(Top action) / 트렌드(현재 vs 직전 24h)

```sql
-- OLTP (9-CTE 골격)
WITH curr_msg AS (
  -- 현재 24h messages: dept별 status별 카운트
  SELECT da.dept_id,
         COUNT(*) FILTER (WHERE m.error IS NOT NULL) AS errors,
         COUNT(*) AS total
  FROM messages m
  JOIN dept_apps da ON m.app_id = da.app_id
  WHERE m.invoke_from != 'debugger'
    AND m.created_at > NOW() - INTERVAL '24 hours'
  GROUP BY da.dept_id
),
curr_wf AS (
  -- 현재 24h workflow_runs: 동일 패턴
  SELECT da.dept_id,
         COUNT(*) FILTER (WHERE w.status = 'failed') AS errors,
         COUNT(*) AS total
  FROM workflow_runs w
  JOIN apps a ON w.app_id = a.id AND a.mode = 'workflow'
  JOIN dept_apps da ON w.app_id = da.app_id
  WHERE w.triggered_from = 'app-run'
    AND w.created_at > NOW() - INTERVAL '24 hours'
  GROUP BY da.dept_id
),
curr_agg AS (
  -- FULL OUTER JOIN msg + wf
  SELECT COALESCE(m.dept_id, w.dept_id) AS dept_id,
         COALESCE(m.errors,0) + COALESCE(w.errors,0) AS errors,
         COALESCE(m.total,0)  + COALESCE(w.total,0)  AS total
  FROM curr_msg m FULL OUTER JOIN curr_wf w USING (dept_id)
),
prev_msg AS (/* 직전 24h messages — curr_msg와 같은 구조 */),
prev_wf AS (/* 직전 24h workflow_runs */),
prev_agg AS (/* msg + wf FULL OUTER JOIN */),
error_causes AS (
  -- 부서×에러 메시지별 카운트 + ROW_NUMBER로 Top 1
  SELECT da.dept_id, m.error AS cause, COUNT(*) AS cnt,
         ROW_NUMBER() OVER (PARTITION BY da.dept_id ORDER BY COUNT(*) DESC) AS rn
  FROM messages m
  JOIN dept_apps da ON m.app_id = da.app_id
  WHERE m.error IS NOT NULL
    AND m.invoke_from != 'debugger'
    AND m.created_at > NOW() - INTERVAL '24 hours'
  GROUP BY da.dept_id, m.error
),
top_causes AS (
  SELECT dept_id, cause FROM error_causes WHERE rn = 1
)
SELECT d.name AS dept,
       c.errors,
       (c.errors * 100.0 / NULLIF(c.total, 0)) AS rate,
       t.cause AS top_cause,
       ((c.errors - p.errors) * 100.0 / NULLIF(p.errors, 0)) AS trend
FROM departments d
LEFT JOIN curr_agg c ON c.dept_id = d.id
LEFT JOIN prev_agg p ON p.dept_id = d.id
LEFT JOIN top_causes t ON t.dept_id = d.id;

-- audit (분모/분자 모두 부정확)
-- 분모(전체): workflow_execute + message_send 합산하려는데
--   workflow audit엔 triggered_from 없어서 디버거 섞임
-- 분자(실패): status='failed' 카운트는 OK
-- 주원인 컬럼: audit의 action 자체가 정규 분류라 OLTP보다 명확
--   → "이 부서는 auth_failed가 주원인", "이 부서는 model_timeout이 주원인" 식
```

**차이**: ❌ **audit 단독 불가** — 분모(전체 호출)에 디버거 데이터 섞임. 단 **주원인 컬럼은 audit이 우월** (자유 텍스트 vs 정규 action) → **하이브리드 후보**: 분모/분자는 OLTP, 주원인 컬럼만 audit 보강.

---

## 종합 매트릭스

| #   | 컴포넌트                     | OLTP | audit | 권장               | 사유                                                           |
| --- | ------------------------ | ---- | ----- | ---------------- | ------------------------------------------------------------ |
| 1   | dept-cumulative          | ✅    | ❌     | OLTP             | 90일 보존 한계 — 누적 카운트 부정확                                       |
| 2   | top-owners               | ✅    | ⚠️    | OLTP             | 90일 한계 + 부서 매핑 추가 JOIN                                       |
| 3   | dept-new-creations-table | ✅    | ✅     | OLTP             | 기간 내라 둘 다 OK. OLTP가 부서 매핑 자연                                 |
| 4   | dept-dau                 | ✅    | ⚠️    | OLTP             | jsonb 추출 비용 + 형변환                                            |
| 5   | model-users              | ✅    | ⚠️    | OLTP             | jsonb GROUP BY 표현식 인덱스 필요                                    |
| 6   | dept-user-activity-table | ✅    | ⚠️    | OLTP             | 신규 판별 90일 한계                                                 |
| 7   | dept-call-count          | ✅    | ❌     | OLTP             | **workflow audit엔 `triggered_from` 누락 → 디버거 데이터 섞여 카운트 부정확** |
| 8   | model-call-share         | ✅    | ⚠️    | OLTP             | messages만 — jsonb GROUP BY 비용. workflow는 양쪽 모두 model 정보 없음   |
| 9   | dept-call-rps-table      | ✅    | ❌     | OLTP             | #7과 동일 + 7-CTE × jsonb/형변환 누적                                |
| 10  | dept-error-rate          | ✅    | ❌     | OLTP             | workflow audit 디버거 분리 불가 → 분모(전체 호출) 부정확                     |
| 11  | **top-error-types**      | ⚠️   | 🌟    | **audit**        | OLTP error 자유 텍스트 vs audit action 정규화                        |
| 12  | dept-error-table         | ✅    | ❌     | OLTP (+audit 보강) | #7·#10과 동일 분모 문제. 주원인 컬럼만 audit 보강 후보                        |

## 핵심 결론

### messages 계열은 audit으로도 그릴 수 있음 — 차이는 비용
- model/token/error 모두 `messages` collector details에 박혀있음
- 그러나 **jsonb 추출 + 형변환 + 큰 테이블** 비용으로 OLTP 대비 2~5배 무거움
- 인덱스 추가로 따라잡을 수 있으나 동적 차원마다 표현식 인덱스 박는 부담

### workflow 계열은 audit 결정적 누락 (4개) ⭐
- `workflow-runs` collector details에 `triggered_from` / `invoke_from` 빠져있음
- **H-DASH-03 디버거 필터 불가** → 디버거 호출이 정상 호출과 섞여 카운트 부정확
- 영향 받는 차트:
  - dept-call-count (분자 부정확)
  - dept-call-rps-table (분자 부정확)
  - dept-error-rate (분모 부정확)
  - dept-error-table (분모 부정확)
- → audit으로 이 4개 그리려면 **collector 수정해서 details에 `triggered_from` 추가** 필요

### audit 보존 기간이 결정적인 케이스 (3개)
- dept-cumulative / top-owners / dept-user-activity 신규 판별
- → 90일 보존 늘리면 OLTP 따라잡지만 스토리지 비용 증가

### audit이 명확히 우월한 1개
- **top-error-types** — OLTP `messages.error` 자유 텍스트 분류 불가 vs audit `action` 이미 정규화

### 부서 매핑 의미 차이 주의
- OLTP: **앱(resource_ownership)** 기준 → "이 앱은 어느 부서 소유인가"
- audit: **사용자(actor_id → department_members)** 기준 → "이 호출을 한 사람이 어느 부서인가"
- 두 개가 다른 의미 — 화면 의도에 따라 선택

→ 우리 화면 설계는 **앱 소유 부서** 기준 (resource_ownership) → OLTP가 자연. audit으로 가려면 매핑 개념을 바꿔야 함.

### Collector 페이로드 풍부도 비교 (13종 전수 점검, 2026-05-08)

| Collector | 충실도 | details 페이로드 | 누락/메모 | 현 12종 영향 |
|---|---|---|---|---|
| `messages.ts` | ✅ 거의 완벽 | model/token/error/invokeFrom/totalPrice 등 14필드 | 없음 | 영향 큼 (KPI/users/calls/errors 다수) |
| `workflow-runs.ts` | ⚠️ **빈약** | id/elapsedTime/totalTokens/error만 5필드 | **`triggered_from`** ⭐ / `invoke_from` / `total_price` 누락 | 영향 큼 — call/error 4개 차트 카운트 부정확 |
| `app-changes.ts` | ✅ 충분 | mode/description/icon/createdAt | 없음 | objects 메트릭 (create 이벤트 활용 가능) |
| `datasets.ts` | △ 보통 | description/indexingTechnique만 | document 수 / created_by 같은 거 없음 | objects 메트릭 (create 이벤트만 활용) |
| `documents.ts` | ✅ 풍부 | datasetId/indexingStatus/error/tokens/dataSourceType | 없음 | 현 12종에 문서 단위 차트 없음 → 영향 X |
| `members.ts` | △ 보통 | tenantId/accountEmail/role | department_id 없음 | 현 12종에 멤버 변경 차트 없음 → 영향 X |
| `conversations.ts` | ✅ 충분 | conversationId/invokeFrom/sessionId | 없음 | 현 12종에 conversation 단위 차트 없음 → 영향 X |
| `message-feedbacks.ts` | ✅ 풍부 | rating/feedbackContent/messageId/appId | 없음 | 현 12종에 없음. 추가 차트 후보 (좋아요/싫어요) |
| `workflow-nodes.ts` | ✅ 풍부 | nodeType/elapsedTime/error/**triggeredFrom** ⭐ | 없음 | 현 12종에 노드 차트 없음. 내부 로그 영역 |
| `workflow-publishes.ts` | ✅ 충분 | workflowId/appId/version/markedName | 없음 | 현 12종에 없음. 거버넌스 이력 후보 |
| `prompt-changes.ts` | △ 보통 | promptPreview(200자)/promptLength | 전체 텍스트 없음 (200자 trim) | 현 12종에 없음. 거버넌스 이력 후보 |
| `provider-changes.ts` | ✅ 충분 | providerName/modelName/modelType/isValid | 없음 | 현 12종에 없음. 거버넌스 이력 후보 |
| `api-tokens.ts` | △ 보통 | tokenType/appId만 | 토큰 사용량/발급자 없음 | 현 12종에 없음. 거버넌스 이력 후보 |

### 일관성 결함 발견 ⭐

**`workflow-nodes.ts`엔 `triggeredFrom` 박혀있는데 `workflow-runs.ts`엔 빠짐.**
- 같은 워크플로우 실행을 다른 입도로 기록하는 두 collector인데 필터 컬럼이 비대칭
- 의도일 수도 있지만 실수일 가능성 높음 (둘 다 워크플로우 실행 메타니까)
- → 승랑님께 제안 시 "node에 있는데 run엔 없네요?" 한 줄로 짧게

### 정리 — 결정적 누락은 1개뿐

13종 collector 점검 결과:
- **결정적 누락**: `workflow-runs.ts` `triggered_from` 1개 (현 12종 4개 차트 영향)
- **그 외**: 현 12종 영향 없음 / 거버넌스 추가 차트 후보로 풍부함
- **추가 차트 후보 영역의 audit 활용도는 매우 높음**:
  - 좋아요/싫어요 (message-feedbacks 풍부)
  - 거버넌스 이력 (prompt/provider/api-tokens/workflow-publishes)
  - 인덱싱 실패 추적 (documents)

→ 현 설계 대시보드는 OLTP 11 + audit 1 결정 그대로 유지 권장. 추가 차트 후보로는 audit 활용 가치 큼.

## 회의 답변 ① 초안

> 12종 중 11개는 OLTP, 1개(top-error-types)는 audit 권장.
>
> 이유 4가지:
> ① **`messages` collector**: details jsonb에 token/model/error 다 박혀있어서 기술적으론 그릴 수 있으나 jsonb 추출 + 형변환 + 큰 테이블 비용으로 OLTP 대비 2~5배 무거움
> ② **`workflow-runs` collector**: details에 `triggered_from`이 빠져있어서 H-DASH-03 디버거 필터 불가 → workflow 계열 4개 차트(call-count/rps/error-rate/error-table)는 카운트 부정확. collector 수정 없이는 audit 사용 불가.
> ③ **누적 카운트(objects 메트릭)**: audit 90일 보존 한계로 부적절
> ④ **부서 매핑 의미 차이**: OLTP=앱 소유 부서 vs audit=사용자 소속 부서. 우리 화면은 앱 기준이라 OLTP가 자연스러움
>
> 단 top-error-types만 audit이 명확히 우월 — `messages.error` 자유 텍스트 vs `audit.action` 정규 분류.

## 미해결 / 후속 검토

- [x] ~~`workflow-runs` collector의 details 페이로드 확인~~ → 완료. `triggered_from`/`invoke_from` 누락 확인
- [x] ~~다른 collector OLTP 컬럼 누락 점검~~ → 완료. 13종 전수 점검 결과 결정적 누락은 `workflow-runs.ts` 1건만
- [ ] **승랑님께 제안할 collector 보강 2건**:
  - `workflow-runs.ts` details에 `triggered_from` / `invoke_from` / `total_price` 추가 (H-DASH-03 호환)
  - 일관성 메모: `workflow-nodes.ts`엔 `triggeredFrom` 박혀있는데 `workflow-runs.ts`엔 빠진 비대칭 — 의도 확인
- [ ] audit 폴링 5분 → 1분/실시간 단축 시 dify-audit 컨테이너 부하 측정
- [ ] dept-error-table "주원인" 컬럼 audit 보강 PoC (하이브리드)
- [ ] audit `(action, occurred_at, tenant_id)` 외 표현식 인덱스 도입 비용 측정
