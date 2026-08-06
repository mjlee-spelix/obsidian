---
tags: [프로젝트, dify, AI-Agent, references]
type: references
date: 2026-05-12
external_dependency_observed_at: 2026-05-12
purpose: 오브젝트 차트 3종이 RBAC만으로 구현 가능한지 검증 결과
related: [rbac-schema.md, design.md, defect-catalog.md#H-DASH-04]
---

# 오브젝트 차트 3종 — RBAC 구현 가능성 검증

> **결론: 3종 모두 RBAC 테이블(`spx_resource_ownership` + `spx_departments` + `spx_department_members` + `spx_accounts`)만으로 구현 가능**.
> audit 테이블이나 Dify OLTP(apps/datasets) JOIN 없이도 동작. 단, 운영 환경의 레거시 앱 미등록 시 H-DASH-04 방어 필수.

## 검증 환경

- DB: `docker compose exec db_postgres psql -U postgres -d dify`
- 데이터: mock fixture 적용 (rbac_mock.sql + oltp_mock.sql)
- 시점: 2026-05-12

---

## 1. dept-cumulative (부서별 누적 오브젝트)

### 목적
부서별 앱/데이터셋/도구 누적 보유 수를 stacked bar chart로 표시.

### SQL 윤곽

```sql
SELECT
  COALESCE(d.name, '미배정') AS department_name,
  ro.resource_type,
  COUNT(*) AS cnt
FROM spx_resource_ownership ro
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE ro.tenant_id = :tenant_id
GROUP BY d.name, ro.resource_type
ORDER BY department_name, resource_type;
```

**사용 테이블**: `spx_resource_ownership` + `spx_departments` (2종)
**Dify OLTP JOIN 불필요**: spx_resource_ownership 자체에 resource_type + resource_id가 있어 apps/datasets/tools 테이블 불필요.

### 실데이터 결과 (14행)

| department_name | resource_type | cnt |
|-----------------|---------------|----:|
| IT 본부 | app | 4 |
| IT 본부 | dataset | 1 |
| IT 본부 | tool | 3 |
| 마케팅 | app | 3 |
| 마케팅 | dataset | 1 |
| 마케팅 | tool | 2 |
| 미배정 | app | 1 |
| 미배정 | dataset | 1 |
| 미배정 | tool | 2 |
| 인사 | app | 1 |
| 인사 | dataset | 1 |
| 재무 | app | 2 |
| 재무 | dataset | 1 |
| 재무 | tool | 1 |

### H-DASH-04 동작
- `owner_department_id IS NULL` → COALESCE → "미배정" 4건 정상 fallback
- **운영 주의**: 레거시 앱이 `spx_resource_ownership`에 행 자체가 없을 경우, 이 쿼리에서 **완전 누락**됨. 운영 시 `apps LEFT JOIN spx_resource_ownership` 패턴으로 교체 필요 (누락 0건 보장):

```sql
-- 운영 방어 버전 (레거시 앱 포함)
SELECT
  COALESCE(d.name, '미배정') AS department_name,
  COALESCE(ro.resource_type, 'app') AS resource_type,
  COUNT(*) AS cnt
FROM apps a
LEFT JOIN spx_resource_ownership ro
  ON ro.resource_id = a.id AND ro.resource_type = 'app' AND ro.tenant_id = a.tenant_id
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE a.tenant_id = :tenant_id
GROUP BY d.name, COALESCE(ro.resource_type, 'app')
-- dataset/tool도 별도 UNION ALL
```

### 구현 가능 여부: **가능** ✅

---

## 2. top-owners (Top 소유자)

### 목적
개인별 오브젝트 보유 수 Top N 표시 (이름 + 부서 + 건수).

### SQL 윤곽

```sql
SELECT
  COALESCE(a.name, '(알 수 없음)') AS owner_name,
  COALESCE(d.name, '미배정') AS department_name,
  COUNT(*) AS object_count
FROM spx_resource_ownership ro
LEFT JOIN spx_accounts a ON a.id = ro.owner_account_id
LEFT JOIN spx_department_members dm
  ON dm.account_id = ro.owner_account_id AND dm.is_active = true
LEFT JOIN spx_departments d ON d.id = dm.department_id
WHERE ro.owner_account_id IS NOT NULL
  AND ro.tenant_id = :tenant_id
GROUP BY a.name, d.name
ORDER BY object_count DESC
LIMIT :top_n;
```

**사용 테이블**: `spx_resource_ownership` + `spx_accounts` + `spx_department_members` + `spx_departments` (4종)
**핵심 컬럼**: `owner_account_id` (NOT `created_by`)

### 실데이터 결과 (10행)

| owner_name | department_name | object_count |
|------------|-----------------|-------------:|
| 신예린 | 재무 | 3 |
| 임지현 | 마케팅 | 3 |
| 김철수 | IT 본부 | 2 |
| 박민수 | IT 본부 | 2 |
| 이영희 | IT 본부 | 2 |
| 장현우 | 마케팅 | 2 |
| 양시우 | 인사 | 1 |
| (알 수 없음) | IT 본부 | 1 |
| 조은서 | 인사 | 1 |
| 윤서연 | 마케팅 | 1 |

### 주의사항

1. **`owner_account_id` vs `created_by` 선택**:
   - `created_by`: 매핑률 29% (mock UUID 문제). 운영에서도 RBAC UI 생성자 ≠ 실제 소유자일 수 있음.
   - `owner_account_id`: 매핑률 94.7% (18/19). **소유자 의미상 적합 → 이쪽 사용**.
   - `owner_account_id IS NULL`인 5건(부서 소유/미배정)은 WHERE 절에서 제외 — 부서 소유 리소스는 dept-cumulative에서 커버.

2. **`(알 수 없음)` 1건**: mock fixture `549120df-...` UUID가 accounts에 없음. 운영에서는 발생 안 할 것. COALESCE 방어.

3. **외부 사용자(end_user) 혼입 없음**: resource_ownership.owner_account_id/created_by는 콘솔 사용자(accounts)만 들어감 — RBAC UI가 세션 account_id 기반이므로 end_user 진입 경로 없음.

### 구현 가능 여부: **가능** ✅

---

## 3. dept-new-creations-table (부서별 신규 생성)

### 목적
선택 기간 내 부서별 신규 생성 오브젝트 수 (resource_type별 breakdown).

### SQL 윤곽

```sql
SELECT
  COALESCE(d.name, '미배정') AS department_name,
  ro.resource_type,
  COUNT(*) AS new_count
FROM spx_resource_ownership ro
LEFT JOIN spx_departments d ON d.id = ro.owner_department_id
WHERE ro.tenant_id = :tenant_id
  AND ro.created_at >= :period_start
  AND ro.created_at < :period_end
GROUP BY d.name, ro.resource_type
ORDER BY department_name, resource_type;
```

**사용 테이블**: `spx_resource_ownership` + `spx_departments` (2종)
**기간 필터**: `created_at` 기반 — resource_ownership 행 생성 = 소유권 등록 시점.

### 실데이터 결과 (30일 기간, 14행)

dept-cumulative와 동일 결과 (mock 데이터가 모두 5월 일괄 INSERT이므로).

### 주의사항

1. **`created_at`의 의미**: resource_ownership의 created_at = **소유권 등록 시점**. apps.created_at(앱 생성 시점)과 다를 수 있음.
   - 운영 시나리오: 앱 먼저 만들고 → 나중에 부서 배정 → resource_ownership.created_at이 뒤늦게 찍힘.
   - **"신규 생성"의 정의가 "소유권 등록 시점"인지 "앱 생성 시점"인지 PM 확인 필요**.
   - 소유권 등록 시점 기준이면 RBAC만으로 충분. 앱 생성 시점이면 `apps.created_at` JOIN 필요 (Dify OLTP 의존 추가).

2. **H-DASH-04 동작**: dept-cumulative와 동일 — COALESCE 미배정 fallback 정상.

3. **인덱스**: `resource_ownership_owner_department_id_idx` 있지만 `created_at` 범위 쿼리용 인덱스 없음. 데이터 소량이면 문제 없지만, 운영 대규모 시 `(tenant_id, created_at)` 인덱스 추가 검토.

### 구현 가능 여부: **가능** ✅ (소유권 등록 기준. 앱 생성 기준이면 apps JOIN 추가 필요)

---

## 4. 종합 — RBAC만으로 가능 여부

### 구현 가능성 매트릭스

| 차트 | RBAC만 가능? | 사용 테이블 | 추가 의존 | 비고 |
|------|:-----------:|-----------|----------|------|
| dept-cumulative | ✅ 가능 | resource_ownership, departments | 없음 (운영: apps LEFT JOIN 권장) | |
| top-owners | ✅ 가능 | resource_ownership, accounts, department_members, departments | 없음 | owner_account_id 기준 |
| dept-new-creations | ✅ 가능 | resource_ownership, departments | 없음 (앱 생성 기준 시 apps 추가) | PM 확인: created_at 의미 |

### 공통 방어 사항

| ID | 패턴 | 적용 |
|----|------|------|
| H-DASH-04 | 미배정 fallback | LEFT JOIN + COALESCE('미배정'). 운영: apps LEFT JOIN 패턴 |
| H-DASH-13 | RBAC 스키마 변경 | SQLAlchemy 모델 클래스 경유 |
| H-DASH-17 | 관찰 부족 | `external_dependency_observed_at: 2026-05-12` 기록 |

### 발견된 데이터 패턴 함정 (H-CAND 후보)

| 후보 ID | 패턴 | 설명 |
|---------|------|------|
| CAND-created-by-mismatch | created_by ≠ 실 accounts | mock fixture에서 단일 UUID 사용. 운영에서는 RBAC UI 세션이 accounts.id 채우므로 괜찮을 것이나, **top-owners는 owner_account_id 사용 권장** (created_by는 "등록한 사람"이지 "소유자"가 아닐 수 있음) |
| CAND-ownership-created-at-semantics | created_at = 소유권 등록 ≠ 앱 생성 | dept-new-creations에서 "신규"의 의미가 모호. PM 확인 전까지 소유권 등록 시점 기준으로 구현 |
| CAND-legacy-app-total-miss | resource_ownership 미등록 앱 | Mock에서 0건이라 안 보이지만 운영에서 다수 예상. dept-cumulative/top-owners에서 누락 → 전체 오브젝트 수 < KPI 카드 총 오브젝트 수 불일치 발생 가능 |

### design.md § 2 반영안 (초안)

오브젝트 차트 3종은 **audit 불필요, RBAC 직접 조회**:

```
데이터 소스 분류:
- KPI/호출/에러/사용자 → audit + RBAC JOIN (마트 후보)
- 오브젝트 3종 → RBAC 직접 조회 (state 본질)
- 모델별 토큰 → OLTP messages 직접 (마트 후보)
```

오브젝트는 "현재 상태(state)"를 보는 것이므로 이벤트 로그(audit)가 아닌 현재 소유 테이블(resource_ownership) 직접 조회가 맞음.

## 관련 노트

- [[3. 프로젝트/spx-agent/references/rbac-schema.md]] — 실데이터 분포 (2026-05-12 갱신)
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] — H-DASH-04, H-DASH-13, H-DASH-17
- [[3. 프로젝트/spx-agent/hdd/design.md]] — § 2 마트 설계
