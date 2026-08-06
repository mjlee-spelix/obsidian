---
tags: [프로젝트, dify, AI-Agent, HDD, infra, references]
type: reference
date: 2026-05-08
last_updated: 2026-05-08
related_design: 3. 프로젝트/spx-agent/hdd/specs/design/data-mart.md
---

# 대시보드 쿼리 인벤토리 (1단계 산출물)

← [[3. 프로젝트/spx-agent/hdd/specs/design/data-mart.md|design]] 의 1단계 산출물

> 작성자: Claude Code (VSCode) · 코드 변경 없음 · 코드에 박힌 그대로 옮김
> 작성일: 2026-05-08
>
> 🔴 **2026-06-09 해석 주의**: 본 문서는 5/8 코드 스냅샷이라 RBAC 테이블을 무접두(`resource_ownership` / `departments` / `department_members`)로 표기함. **2026-05-19 마이그레이션 이후 실제 테이블명은 전부 `spx_` 접두사**(`spx_resource_ownership` / `spx_departments` / `spx_department_members` / `spx_resource_permissions` / `spx_rbac_audit_logs`, `accounts`→`spx_accounts`). 아래 표의 무접두 RBAC/accounts 명은 모두 `spx_` 접두사로 해석할 것. 근거: `references/rbac-schema.md` 🔴 배너.

## 1. 요약

| 항목 | 값 |
|------|-----|
| 총 엔드포인트 수 | **15** (Phase 1: 4, Phase 2 drill-through: 11) + mock drawer 1 |
| 사용 테이블 종류 수 | **8** (OLTP 5 + RBAC 3) |
| 가장 많이 쓰이는 테이블 Top 3 | `messages` (7개 서비스) · `resource_ownership` (6개) · `departments` (6개) |
| 가장 복잡한 쿼리 | `drill_calls.get_dept_call_rps_table` — CTE 7개, UNION ALL, ROW_NUMBER 윈도우 함수, 현재+이전 기간 이중 집계 |
| Silent fallback 의심 | **0건** — 모든 `except Exception` 블록에 `logger.exception()` 확인됨 |

---

## 2. 엔드포인트별 쿼리 인벤토리 표

### Phase 1 — 메인 대시보드

| # | 컴포넌트 | 라우트 | 서비스 파일 | 함수명 | 주 쿼리 (요약) | 소스 테이블 | RBAC 의존 | GROUP BY | WHERE 핵심 | JOIN | 비고 |
|---|----------|--------|-------------|--------|----------------|-------------|-----------|----------|------------|------|------|
| 1 | KPI 카드: 총 오브젝트 | `GET /admin/dashboard/kpi` | `dashboard_kpi_service.py` | `_get_total_objects` | COUNT(apps) + COUNT(datasets) + COUNT(resource_ownership WHERE tool) | `apps`, `datasets`, `resource_ownership` | resource_ownership | — | `resource_type='tool'`, `created_at BETWEEN` (증감용) | — | 증감률: 현재/이전 기간 new objects 비교 |
| 2 | KPI 카드: 활성 사용자 | `GET /admin/dashboard/kpi` | `dashboard_kpi_service.py` → `query_helpers.py` | `_get_active_users` → `count_active_users_union` | UNION(messages.from_end_user_id, workflow_runs.created_by) DISTINCT | `messages`, `workflow_runs` | — | — | `invoke_from != 'debugger'`, `triggered_from = 'app-run'`, `from_end_user_id IS NOT NULL`, `created_by IS NOT NULL` | `workflow_runs JOIN apps (mode='workflow')` | H-DASH-01 ✅ H-DASH-03 ✅ H-DASH-08 ✅ |
| 3 | KPI 카드: API 호출 | `GET /admin/dashboard/kpi` | `dashboard_kpi_service.py` → `query_helpers.py` | `_get_api_calls` → `count_messages` + `count_workflow_runs` | COUNT(messages) + COUNT(workflow_runs) | `messages`, `workflow_runs`, `apps` | — | — | `invoke_from != 'debugger'`, `triggered_from = 'app-run'`, `mode = 'workflow'` | `workflow_runs JOIN apps` | H-DASH-01 ✅ H-DASH-03 ✅ |
| 4 | KPI 카드: 에러율 24h | `GET /admin/dashboard/kpi` | `dashboard_kpi_service.py` → `query_helpers.py` | `_get_error_rate_24h` | (error_msgs + failed_wf) / (total_msgs + total_wf) × 100 — 24h vs 이전 24h | `messages`, `workflow_runs`, `apps` | — | — | `error IS NOT NULL` (msg), `status='failed'` (wf), 24h/48h 윈도우 | `workflow_runs JOIN apps` | H-DASH-01 ✅ H-DASH-03 ✅, H-DASH-07 분모 0 보호: 앱 코드 |
| 5 | 부서별 오브젝트 | `GET /admin/dashboard/dept-objects` | `dashboard_dept_objects_service.py` | `_get_dept_counts` + `_get_unassigned_counts` | 부서별 resource_type별 COUNT + 미배정 NOT EXISTS 카운트 | `departments`, `resource_ownership`, `apps`, `datasets` | departments, resource_ownership | `d.id, d.name, o.resource_type` | `d.is_active = true` | `resource_ownership JOIN departments` | H-DASH-04 ✅ 미배정 fallback |
| 6 | 모델별 토큰 | `GET /admin/dashboard/model-tokens` | `dashboard_model_tokens_service.py` → `query_helpers.py` | `get_model_tokens` → `sum_message_tokens` + `sum_workflow_tokens` | SUM(message_tokens + answer_tokens) GROUP BY model + COALESCE(SUM(wf.total_tokens), 0) | `messages`, `workflow_runs`, `apps` | — | `model_provider, model_id` | `invoke_from != 'debugger'`, `triggered_from = 'app-run'`, `mode = 'workflow'` | `workflow_runs JOIN apps` | H-DASH-01 ✅ H-DASH-03 ✅, wf 토큰은 모델 미분류(H-DASH-02) |
| 7 | 부서별 활동 테이블 | `GET /admin/dashboard/dept-activity` | `dashboard_dept_activity_service.py` | `_get_dept_activity` + `_get_unassigned_activity` | 6-CTE 복합: dept_apps → msg/wf_calls → msg/wf_tokens → LEFT JOIN departments | `departments`, `resource_ownership`, `messages`, `workflow_runs`, `apps` | departments, resource_ownership | `owner_department_id, resource_type` / `da.department_id` | `d.is_active = true`, `invoke_from != 'debugger'`, `triggered_from = 'app-run'`, `mode = 'workflow'` | CTE dept_apps + 6× LEFT JOIN | H-DASH-01 ✅ H-DASH-03 ✅ H-DASH-04 ✅ 가장 복잡한 Phase 1 쿼리 |

### Phase 2 — Drill-Through: Objects

| # | 컴포넌트 | 라우트 | 서비스 파일 | 함수명 | 주 쿼리 (요약) | 소스 테이블 | RBAC 의존 | GROUP BY | WHERE 핵심 | JOIN | 비고 |
|---|----------|--------|-------------|--------|----------------|-------------|-----------|----------|------------|------|------|
| 8 | 부서별 누적 오브젝트 | `GET /admin/dashboard/drill/objects/dept-cumulative` | `dashboard_drill_objects_service.py` | `get_dept_cumulative` | COALESCE(d.name, '미배정'), COUNT(*) | `resource_ownership`, `departments` | 둘 다 | `d.name` | — | `LEFT JOIN departments` | H-DASH-04 ✅, H-DASH-01 N/A, H-DASH-03 N/A |
| 9 | Top 소유자 10명 | `GET /admin/dashboard/drill/objects/top-owners` | `dashboard_drill_objects_service.py` | `get_top_owners` | accounts.name + dept.name + COUNT(*) LIMIT 10 | `resource_ownership`, `accounts`, `department_members`, `departments` | 3개 | `a.name, d.name` | `owner_account_id IS NOT NULL`, `dm.is_active = true` | `JOIN accounts` + `LEFT JOIN department_members` + `LEFT JOIN departments` | H-DASH-04 ✅, LIMIT 10 |
| 10 | 부서별 신규 생성 | `GET /admin/dashboard/drill/objects/dept-new-creations` | `dashboard_drill_objects_service.py` | `get_dept_new_creations` | COUNT(*) FILTER(resource_type) GROUP BY dept | `resource_ownership`, `departments` | 둘 다 | `d.name` | `created_at BETWEEN` | `LEFT JOIN departments` | H-DASH-04 ✅, FILTER 절 사용 |

### Phase 2 — Drill-Through: Users

| # | 컴포넌트 | 라우트 | 서비스 파일 | 함수명 | 주 쿼리 (요약) | 소스 테이블 | RBAC 의존 | GROUP BY | WHERE 핵심 | JOIN | 비고 |
|---|----------|--------|-------------|--------|----------------|-------------|-----------|----------|------------|------|------|
| 11 | 모델별 사용자 수 | `GET /admin/dashboard/drill/users/model-users` | `dashboard_drill_users_service.py` | `get_model_users` | COUNT(DISTINCT from_end_user_id) GROUP BY model_id | `messages` | — | `m.model_id` | `invoke_from != 'debugger'`, `from_end_user_id IS NOT NULL` | — | H-DASH-03 ✅ H-DASH-08 ✅ |
| 12 | 부서별 DAU | `GET /admin/dashboard/drill/users/dept-dau` | `dashboard_drill_users_service.py` | `get_dept_dau` | COUNT(DISTINCT from_end_user_id) GROUP BY dept | `messages`, `department_members`, `departments` | department_members, departments | `d.name` | `invoke_from != 'debugger'`, `from_end_user_id IS NOT NULL` | `LEFT JOIN department_members` + `LEFT JOIN departments` | H-DASH-03 ✅ H-DASH-04 ✅ H-DASH-08 ✅ |
| 13 | 부서별 사용자 활동 상세 | `GET /admin/dashboard/drill/users/dept-user-activity` | `dashboard_drill_users_service.py` | `get_dept_user_activity` | 5-CTE: period_users → prev_users → dau_wau → new_users → churned | `messages`, `department_members`, `departments` | department_members, departments | `d.name, m.from_end_user_id` 등 다단계 | `invoke_from != 'debugger'` ×3, `from_end_user_id IS NOT NULL` ×3, `NOT EXISTS` (신규/이탈 판별) | `LEFT JOIN department_members` + `LEFT JOIN departments` ×2 | H-DASH-03 ✅ H-DASH-04 ✅ H-DASH-08 ✅, CASE WHEN 윈도우 DAU/WAU |

### Phase 2 — Drill-Through: Calls

| # | 컴포넌트 | 라우트 | 서비스 파일 | 함수명 | 주 쿼리 (요약) | 소스 테이블 | RBAC 의존 | GROUP BY | WHERE 핵심 | JOIN | 비고 |
|---|----------|--------|-------------|--------|----------------|-------------|-----------|----------|------------|------|------|
| 14 | 모델별 호출 점유율 | `GET /admin/dashboard/drill/calls/model-call-share` | `dashboard_drill_calls_service.py` | `get_model_call_share` | COUNT(*) GROUP BY COALESCE(model_id, '미분류') | `messages` | — | `model_id` | `invoke_from != 'debugger'` | — | H-DASH-02 ✅ H-DASH-03 ✅ |
| 15 | 부서별 호출 수 | `GET /admin/dashboard/drill/calls/dept-call-count` | `dashboard_drill_calls_service.py` | `get_dept_call_count` | 3-CTE: dept_apps + msg_calls + wf_calls → FULL OUTER JOIN | `resource_ownership`, `messages`, `workflow_runs`, `apps`, `departments` | resource_ownership, departments | `d.name` | `invoke_from != 'debugger'`, `triggered_from = 'app-run'`, `mode = 'workflow'` | CTE dept_apps + LEFT JOINs + FULL OUTER JOIN | H-DASH-01 ✅ H-DASH-03 ✅ H-DASH-04 ✅ |
| 16 | 부서별 호출 RPS 테이블 | `GET /admin/dashboard/drill/calls/dept-call-rps` | `dashboard_drill_calls_service.py` | `get_dept_call_rps_table` | **7-CTE**: dept_apps → curr/prev_calls(UNION ALL) → curr/prev_agg → dept_app_calls(ROW_NUMBER) → top_apps | `resource_ownership`, `messages`, `workflow_runs`, `apps`, `departments` | resource_ownership, departments | `d.name`, `d.name + a.name` 다단계 | `invoke_from != 'debugger'` ×3, `triggered_from = 'app-run'` ×2, `mode = 'workflow'` ×2 | CTE 7개 + UNION ALL + LEFT JOINs | H-DASH-01 ✅ H-DASH-03 ✅ H-DASH-04 ✅, **가장 복잡한 쿼리** |

### Phase 2 — Drill-Through: Errors

| # | 컴포넌트 | 라우트 | 서비스 파일 | 함수명 | 주 쿼리 (요약) | 소스 테이블 | RBAC 의존 | GROUP BY | WHERE 핵심 | JOIN | 비고 |
|---|----------|--------|-------------|--------|----------------|-------------|-----------|----------|------------|------|------|
| 17 | 부서별 에러율 | `GET /admin/dashboard/drill/errors/dept-error-rate` | `dashboard_drill_errors_service.py` | `get_dept_error_rate` | 3-CTE: dept_apps + msg_stats(FILTER error) + wf_stats(FILTER failed) → FULL OUTER JOIN | `resource_ownership`, `messages`, `workflow_runs`, `apps`, `departments` | resource_ownership, departments | `d.name` | `invoke_from != 'debugger'`, `triggered_from = 'app-run'`, `mode = 'workflow'`, `status = 'error'/'failed'` | CTE + FULL OUTER JOIN | H-DASH-01 ✅ H-DASH-03 ✅ H-DASH-04 ✅ H-DASH-07 ✅ 분모 0 보호 |
| 18 | 부서별 에러 테이블 | `GET /admin/dashboard/drill/errors/dept-error-table` | `dashboard_drill_errors_service.py` | `get_dept_error_table` | **9-CTE**: curr_msg/wf → curr_agg → prev_msg/wf → prev_agg → error_causes(ROW_NUMBER) → top_causes | `resource_ownership`, `messages`, `workflow_runs`, `apps`, `departments` | resource_ownership, departments | `d.name`, `d.name + m.error` 다단계 | `invoke_from != 'debugger'` ×3, `triggered_from = 'app-run'` ×2, `mode = 'workflow'` ×2, `status = 'error'/'failed'` | CTE 9개 + FULL OUTER JOIN ×2 + ROW_NUMBER | H-DASH-01 ✅ H-DASH-03 ✅ H-DASH-04 ✅ H-DASH-07 ✅, **CTE 수 최다** |

> **참고**: `GET /admin/dashboard/drawer` (Phase 2 chart-drawer)는 현재 mock 데이터 반환 — 서비스 호출 없음, 인벤토리 대상 제외.

---

## 3. 사용 테이블 카탈로그

| 테이블 | 사용 서비스 (파일 수) | 영역 |
|--------|----------------------|------|
| `messages` | kpi, dept_activity, model_tokens, drill_users(3), drill_calls(3), drill_errors(2) — **7개** | OLTP |
| `workflow_runs` | kpi, dept_activity, model_tokens, drill_calls(2), drill_errors(2) — **5개** | OLTP |
| `apps` | kpi, dept_objects, dept_activity, model_tokens, drill_calls(2), drill_errors(2) — **5개** | OLTP |
| `datasets` | kpi, dept_objects — **2개** | OLTP |
| `accounts` | drill_objects(top-owners) — **1개** | OLTP |
| `resource_ownership` | kpi, dept_objects, dept_activity, drill_objects(3), drill_calls(2), drill_errors(2) — **6개** | RBAC |
| `departments` | dept_objects, dept_activity, drill_objects(3), drill_users(2), drill_calls(2), drill_errors(2) — **6개** | RBAC |
| `department_members` | drill_objects(top-owners), drill_users(2) — **2개** | RBAC |

**보조 파일**: `query_helpers.py` — kpi, model_tokens 서비스가 공유하는 7개 헬퍼 함수 (count_messages, count_workflow_runs, count_active_users_union, count_active_users_messages, count_active_users_workflows, sum_message_tokens, sum_workflow_tokens)

---

## 4. Dify upstream 비교

| 비교 항목 | Dify (앱별 통계) | SPX 대시보드 (전사 관리자용) | 영향 분석 |
|-----------|-----------------|---------------------------|-----------|
| **1. 응답 페이로드 크기** | 일자별 GROUP BY → 30~365행 (1행/일). 전체 반환, 페이지네이션 없음 | 부서별/모델별 GROUP BY → 행 수는 부서·모델 수에 비례(10~50행). 단 이전 기간 증감률 포함 시 2× 쿼리 | Dify가 더 크지만 단순; SPX는 행 수 적으나 쿼리 복잡도 높음 |
| **2. 캐싱 레이어** | **없음** — Redis/lru_cache 미사용. 매 요청 DB 직접 | **없음** — 백엔드 캐싱 0. 프론트엔드 TanStack Query staleTime 5분이 유일한 캐시 | **양쪽 모두 캐싱 부재** — SPX의 복합 쿼리에서 더 큰 페널티 |
| **3. 호출 횟수 / 페이지** | 앱 1개 대시보드: 8개 API (chat) / 4개 API (workflow) | 메인 대시보드 진입: **4개 API** 동시. Drill-through 클릭 시 추가 2~3개 | SPX가 호출 수는 적으나, 각 호출의 쿼리 무게가 무거움 |
| **4. 쿼리 단순도** | 단일 테이블 GROUP BY 위주 (`messages` 1개 테이블, JOIN 0~1). Raw SQL text() 사용 | 다중 CTE + 다중 LEFT JOIN + FULL OUTER JOIN + UNION ALL. Raw SQL text() 사용 | **SPX 쿼리 복잡도 ≫ Dify**. 특히 #16 (7-CTE), #18 (9-CTE) |
| **5. 페이지네이션 / LIMIT** | 통계 엔드포인트에 LIMIT 없음 (일자 GROUP BY → 자연 제한). 로그 조회에만 `page + limit=20` | LIMIT 10 = drill_objects.top-owners 1개만. 나머지 전부 무제한 | 대부분 동일 패턴 (GROUP BY = 자연 제한). SPX는 부서 수 증가 시 주의 |

**핵심 차이 요약**:

Dify는 **단일 앱** 범위에서 단순 GROUP BY를 반복하므로 인덱스만으로 빠름. SPX는 **전사 범위**에서 앱→부서 매핑(resource_ownership JOIN) + 2개 소스 테이블(messages + workflow_runs) 합산이 필수라 쿼리 복잡도가 구조적으로 높음.

**응답 시간 느림의 가장 유력한 원인 후보**:
1. 전사 범위 풀스캔 (app_id 필터 없이 messages/workflow_runs 전체 탐색)
2. CTE 다단 결합 (특히 현재+이전 기간 이중 집계 패턴)
3. 백엔드 캐싱 부재 (동일 기간 필터로 반복 요청 시 매번 풀 쿼리)

---

## 5. Silent fallback 위생 점검

**결과: 의심 0건**

모든 서비스 파일의 `except Exception` 블록에 `logger.exception()` 호출 확인됨:

| 서비스 파일 | `except Exception` 수 | `logger.exception` 유무 |
|-------------|----------------------|------------------------|
| `dashboard_kpi_service.py` | 0 | N/A (try-except 없음, 예외 전파) |
| `dashboard_dept_objects_service.py` | 2 | ✅ 2/2 |
| `dashboard_dept_activity_service.py` | 2 | ✅ 2/2 |
| `dashboard_model_tokens_service.py` | 0 | N/A (try-except 없음, 예외 전파) |
| `dashboard_drill_objects_service.py` | 3 | ✅ 3/3 |
| `dashboard_drill_users_service.py` | 3 | ✅ 3/3 |
| `dashboard_drill_calls_service.py` | 3 | ✅ 3/3 |
| `dashboard_drill_errors_service.py` | 2 | ✅ 2/2 |

**주의**: `dashboard_kpi_service.py`와 `dashboard_model_tokens_service.py`는 try-except 자체가 없음 — 예외가 컨트롤러까지 전파됨. 이는 의도적 설계로 보이나, 프로덕션에서 500 에러 노출 가능. 데이터 마트 전환 시 fallback 정책(§ 6.2 옵션 B) 적용 대상.
