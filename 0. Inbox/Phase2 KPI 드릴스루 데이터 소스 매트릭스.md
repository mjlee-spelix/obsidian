---
tags: [프로젝트, dify, AI-Agent, HDD, 임시]
date: 2026-05-06
last_updated: 2026-05-06
status: 검토 후 design.md로 이관 예정
---
# Phase 2 KPI 드릴스루 — 데이터 소스 매트릭스

> **위치**: 임시. 검토 후 [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md]] 의 새 섹션으로 옮길 후보.
> **목적**: 정정된 12종 컴포넌트 (좌4 + 우4 + 표4) 각각이 어떤 테이블/소스를 쓰는지 한눈에. 백엔드(Phase 2 3단계) 작업 시 "어떤 테이블 써?" 매번 헤매지 않게.
> **참조**: [[3. 프로젝트/spx-agent/references/dify-db-schema.md]], [[3. 프로젝트/spx-agent/references/audit-schema.md]], [[3. 프로젝트/spx-agent/references/rbac-schema.md]]
> **2026-05-06 정정**: RBAC 가용성 ❌ → ✅ (5/6 오후 발견 + 권대리님 명명 확정). 컬럼명 sp_ → 회사 표준 (prefix 없음). 자세히 [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|H-DASH-17]] 참조.

## 데이터 소스 분류 — 3개 영역 (2026-05-06 갱신)

| 영역 | 테이블 | 의존 | 가용성 |
|---|---|---|---|
| **Dify OLTP** | `messages`, `workflow_runs`, `apps`, `datasets`, `accounts` | 없음 | ✅ 즉시 |
| **RBAC** (회사 표준 — prefix 없음) | `departments`, `department_members`, `resource_ownership` | 권대리님 (`feat/rbac` 브랜치) | ✅ **즉시** (5/6 우리 docker DB에 영속 잔존 + 권대리님 명명 확정) |
| **Audit Logs** | `audit.audit_events` | 승랑님 — `lib/` 푸시 누락으로 build 보류 | ⏸ **보류** (push 받기 전) |

## 12종 컴포넌트 매트릭스

### objects 메트릭 (총 오브젝트)

| 영역 | 컴포넌트 | 주 테이블 | RBAC 의존 | 즉시 가능? | 비고 |
|---|---|---|---|---|---|
| 좌 | dept-cumulative | `apps` + `datasets` + `tools` COUNT GROUP BY 부서 | resource_ownership + departments | ✅ | H-DASH-04 미배정 fallback (`owner_department_id` nullable) |
| 우 | top-owners | `apps.created_by` + `datasets.created_by` COUNT GROUP BY user | accounts (필수) + department_members/departments (라벨용) | ✅ | account_id 직접 매핑으로 H-DASH-14 자동 해소 |
| 표 | dept-new-creations-table | `apps.created_at` + `datasets.created_at` + `tools.created_at` BETWEEN, GROUP BY 부서·type | resource_ownership + departments | ✅ | 컬럼: 부서/App/KB/Tool/신규 |

### users 메트릭 (활성 사용자)

| 영역 | 컴포넌트 | 주 테이블 | RBAC 의존 | 즉시 가능? | 비고 |
|---|---|---|---|---|---|
| 좌 | dept-dau | `messages.from_end_user_id` + `workflow_runs.created_by` DISTINCT (24h) GROUP BY 부서 | department_members + departments | ✅ | H-DASH-08 NULL 필터. `account_id` 직접 매핑 |
| 우 | model-users | `messages.model_id` + COUNT DISTINCT `from_end_user_id` GROUP BY model_id | **없음** ✨ | ✅ | messages만 |
| 표 | dept-user-activity-table | DAU(24h) + WAU(7d) + 신규(`accounts.created_at`) + 이탈(직전 활성·현 비활성) GROUP BY 부서 | department_members + departments | ✅ | 컬럼: 부서/DAU/WAU/신규/이탈 |

### calls 메트릭 (API 호출)

| 영역 | 컴포넌트 | 주 테이블 | RBAC 의존 | 즉시 가능? | 비고 |
|---|---|---|---|---|---|
| 좌 | dept-call-count | `messages` + `workflow_runs` COUNT (AppMode 분기) GROUP BY 부서 | resource_ownership + departments | ✅ | H-DASH-01/03 |
| 우 | model-call-share | `messages` COUNT GROUP BY `model_id` (AppMode 분기) | **없음** ✨ | ✅ | WORKFLOW는 "미분류" 버킷 (H-DASH-02) |
| 표 | dept-call-rps-table | 호출=COUNT, RPS=COUNT/86400, Top앱=서브쿼리(부서 내 1위), 추세=현재24h vs 직전24h | resource_ownership + departments | ✅ | 컬럼: 부서/호출/RPS/Top앱/추세 |

### errors 메트릭 (에러율 24h)

| 영역 | 컴포넌트 | 주 테이블 | RBAC 의존 | 즉시 가능? | 비고 |
|---|---|---|---|---|---|
| 좌 | dept-error-rate | `messages.status='error'` + `workflow_runs.status='failed'` (24h) / 전체호출 GROUP BY 부서 | resource_ownership + departments | ✅ | |
| 우 | top-error-types | **`audit.audit_events`** action GROUP BY WHERE `status='failed'` ORDER BY count | **없음** ⭐ | ⏸ **보류** | audit build 막힘 (승랑님 lib/ push 대기) |
| 표 | dept-error-table | 실패=COUNT, 에러율=실패/전체×100, 주원인=Top action(서브쿼리), 트렌드=직전24h GROUP BY 부서 | resource_ownership + departments (+ audit_events for 주원인 — 보류) | ✅ (주원인 컬럼은 audit 받기 전엔 messages.error 텍스트 분류 fallback) | 컬럼: 부서/에러/에러율/주원인/트렌드 |

## 즉시 작업 가능 정리 (2026-05-06 갱신 — RBAC 8종 즉시 가능 ✨)

### ✅ 즉시 백엔드 작성 가능 (11종)

**Dify OLTP만 (RBAC 무관, 2종)** — 가장 단순:
```
1. users 우  — model-users      (messages만)
2. calls 우  — model-call-share (messages만, AppMode 분기)
```

**Dify OLTP + RBAC (9종)** — 우리 docker DB에 RBAC 5종 테이블 영속 잔존 + 권대리님 명명 확정:
```
3. objects 좌  — dept-cumulative           (apps + resource_ownership + departments)
4. objects 우  — top-owners                 (accounts + department_members + departments)
5. objects 표  — dept-new-creations-table   (apps/datasets/tools + resource_ownership)
6. users 좌    — dept-dau                   (messages + department_members)
7. users 표    — dept-user-activity-table   (messages + department_members + accounts)
8. calls 좌    — dept-call-count            (messages + workflow_runs + resource_ownership)
9. calls 표    — dept-call-rps-table        (messages + resource_ownership + apps)
10. errors 좌  — dept-error-rate            (messages.status='error' + workflow_runs.status='failed' + resource_ownership)
11. errors 표  — dept-error-table           (errors 좌 + 부서별 Top action)
```

→ **이게 5/6 오후 발견의 핵심 가치**. RBAC 의존 8종이 ❌ → ✅로 전환됨. **김이사님 ETA 문의 불필요해짐**.

### ⏸ 보류 (1종, audit 영역)

```
12. errors 우 — top-error-types  (audit_events 의존, 승랑님 lib/ push 대기)
```

→ 받으면 1시간 안에 추가 작업. 그동안 11종 풀가동.

### 📦 mock 데이터 INSERT 필요 (백엔드 작성 직후)

RBAC 5종 테이블 모두 데이터 0건 → 시각 검증 위해 **fixture INSERT 필수**:
- 부서 5~6개 (`departments`)
- 부서별 사용자 10~30명 (`department_members`)
- 앱/데이터셋/도구의 부서 매핑 (`resource_ownership`)
- 권한 데이터 (`resource_permissions`) — 우리 무관이라 생략 가능

VSCode Claude에 fixture SQL 작성 위임 또는 본인 직접 SQL.

## audit_events 활용 깊이 — top-error-types만 채택한 이유

| 후보 | audit_events vs Dify OLTP 비교 |
|---|---|
| **top-error-types** ⭐ | audit_events 압승. action 분류가 이미 잘 돼있음 (`model_timeout`, `rate_limit`, `auth_failed`). 인덱스 `(action, status)` 활용. messages.error 텍스트 패턴 분류보다 의미 명확 |
| **dept-error 표 "주 원인" 컬럼** | 1차는 messages.error 자체 분류, 시간 되면 audit_events 보강. ⚠️ audit_events.target_id ↔ apps.id 매핑은 `target_type='app'` 가정 검증 필요. 5분 폴링 지연 → KPI #4(실시간) 의미 어긋남 |
| 그 외 11종 | messages/workflow_runs 직접 우위 (AppMode 분기/디버깅 필터/실시간성/에러 의미) |

## 의문점 — audit_events 추가 검증 필요

VSCode Claude에 백엔드 던지기 전 확인:

1. **audit_events.target_type='app' + target_id ↔ apps.id 매핑 정확도** — JOIN 가능한지 샘플 데이터로 확인
2. **details jsonb 키 명세** — model 정보 들어있는지 (model-* 차트 보강 가능성)
3. **5분 폴링 갭 영향** — top-error-types도 24h 윈도우라 5분 갭은 무의미할 듯. 단 "방금 발생한 에러가 안 보임" UX는 H-DASH-06 패턴
4. **tenant_id 필터링** — audit_events에 tenant_id 컬럼 존재 (확인됨, NULL = 글로벌)

## 다음 행동 추천 (2026-05-06 갱신)

1. **이 매트릭스를 design.md 정식 섹션으로 이관** — VSCode Claude가 백엔드 작성 시 매번 참조
2. **VSCode Claude에 11종 백엔드 던지기** (audit 1종 제외 전부):
   ```
   Phase 2 3단계 백엔드 11종 시작 (top-error-types 1종은 audit lib/ push 대기로 보류).
   회사 RBAC 명명 표준은 prefix 없음 — references/rbac-schema.md 갱신 명세 사용.
   - departments / department_members / resource_ownership 활용
   - sp_* 가정 모두 폐기 (H-DASH-17)
   - mock fixture SQL도 같이 작성해서 우리 docker DB의 RBAC 테이블에 INSERT
   SESSION_HISTORY부터 읽어.
   ```
3. **승랑님 lib/ push 받으면** audit 사이클 마무리 + top-error-types 1종 추가 작업
4. ~~**김이사님 RBAC 스키마 ETA 문의**~~ — 5/6 발견으로 불필요 (이미 우리 환경 가용)

## 관련

- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md]] — 이 매트릭스 이관 후보
- [[3. 프로젝트/spx-agent/references/audit-schema.md]] — audit_events 명세
- [[3. 프로젝트/spx-agent/references/dify-db-schema.md]] — messages/workflow_runs 명세
- [[3. 프로젝트/spx-agent/references/rbac-schema.md]] — sp_ 테이블 명세
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] — H-DASH-01/02/03/04/08/14 (관련 결함 패턴)
