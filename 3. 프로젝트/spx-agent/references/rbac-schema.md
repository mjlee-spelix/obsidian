# RBAC 테이블 스키마 (회사 표준 명명 — `spx_` 접두사)

> 🔴 **2026-06-09 전면 정정 (코드/DB SoT 대조)**: 본 문서가 5/6~5/19에 "RBAC 5종 prefix 없음 유지"라고 기술했으나 **실제와 어긋남**. 마이그레이션 `20260519000000_fix_layer1_mview_naming_and_rbac_prefix` 및 `api/models/rbac.py`의 `__tablename__`, 그리고 라이브 DB(`\dt`)를 직접 대조한 결과 **RBAC 5종은 전부 `spx_` 접두사**다.
> - 실제 테이블: `spx_departments` / `spx_department_members` / `spx_resource_ownership` / `spx_resource_permissions` / `spx_rbac_audit_logs`
> - 근거: 마이그레이션 주석 `Unprefixed RBAC refs (...) -> spx_resource_ownership, spx_department_members, spx_departments`; 라이브 DB에 `departments` 등 무접두 테이블은 **존재하지 않음**(`relation "departments" does not exist`).
> - 아래 본문/관계도/매핑표의 무접두 표기는 개념 설명용 shorthand로 남겨두되, **물리 테이블명은 항상 `spx_` 접두사**로 해석할 것. 컬럼/제약/구조 자체는 변동 없음.

> 에이전트 참조용. 권대리님이 `feat/rbac` 브랜치에 구현한 RBAC 영역 실제 DB 명세.
> **2026-05-06 전면 정정**: 5/4 작성된 `sp_` prefix 가정이 회사 실제 명명과 어긋남 발견 → 권대리님 명명 확정 답변(2026-05-06 슬랙) 받아 갱신. 정정 사실은 [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|H-DASH-17]] 참조.
> 명세 출처: 본인 docker `db_postgres` 컨테이너의 실 DB `\d <테이블명>` 직접 추출 (5/6 오후, feat/rbac 체크아웃 + `flask db upgrade` 후).
> **`external_dependency_observed_at: 2026-05-12`** — 실데이터 분포 관찰 (H-DASH-17 방어)

> **2026-05-19 보정 — `accounts` → `spx_accounts` rename 반영 (권대리님 5/18 `f88f4a6`)**: ~~RBAC 5종은 prefix 없음 유지~~ → **(2026-06-09 정정) RBAC 5종도 `spx_` 접두사로 통일됨** (위 🔴 배너 참조). `department_members.account_id` / `resource_ownership.owner_account_id` / `resource_ownership.created_by` 등 FK 의도가 참조하는 Dify 테이블 이름이 `accounts` → `spx_accounts`로 변경됨. RBAC 5종 자체 컬럼/제약은 변경 없음 (FK가 명시적 외부 FK가 아닌 의도 매핑이라 rename 영향이 RBAC DDL엔 없음). 본 문서의 쿼리 예시 + 매핑 표가 새 이름 반영.

## 명명 표준 (회사)

- **`spx_` 접두사** — 모든 RBAC 테이블이 `spx_` 접두사 사용 (2026-05-19 마이그레이션으로 통일). ~~`sp_`(5/4 가정)~~ → ~~prefix 없음(5/6 가정)~~ → `spx_`(5/19 최종)
- 부서/멤버: `spx_departments`, `spx_department_members`
- 객체 권한: `spx_resource_*` (일반화된 명명, 'app'/'dataset'/'tool' 모두 한 테이블에 통합)
- 모든 테이블 `tenant_id` UUID 컬럼 포함 (워크스페이스 격리)
- 기본 PK는 모두 `id` (UUID)
- 시각 컬럼: `created_at` / `updated_at` (`assigned_at` 등 의미별 변형)

## 테이블 관계도

```
spx_departments (id PK)
 └── spx_department_members (department_id, account_id) — 다대일 (1 사용자 = 1 부서, UNIQUE)
      └── spx_accounts (Dify 기본 'accounts' 5/18 rename — account_id로 직접 매핑, H-DASH-14 회피 ✅)

spx_resource_ownership (resource_type + resource_id UNIQUE per tenant)
 ├── owner_department_id → spx_departments (nullable — 미배정 가능)
 ├── owner_account_id → spx_accounts (nullable — 사용자 직접 소유 시)
 └── visibility_scope (varchar — private/department/tenant 등)

spx_resource_permissions (RBAC 권한 — 우리 대시보드 무관)
 ├── resource_type + resource_id (대상)
 ├── principal_type + principal_id (권한 받는 주체)
 └── action + effect ('allow'/'deny')

spx_rbac_audit_logs (RBAC 변경 감사 — 우리 대시보드 무관)
```

## 1. `spx_departments` (부서 마스터)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| id | uuid | NOT NULL | | PK |
| tenant_id | uuid | NOT NULL | | 워크스페이스 격리 |
| code | varchar(100) | NOT NULL | | 부서 코드 (예: 'IT', 'MKT'). `(tenant_id, code)` UNIQUE |
| name | varchar(255) | NOT NULL | | 표시명 |
| is_active | boolean | NOT NULL | true | |
| created_by | uuid | NULL | | 생성자 account_id |
| created_at | timestamp | NOT NULL | CURRENT_TIMESTAMP | |
| updated_at | timestamp | NOT NULL | CURRENT_TIMESTAMP | |

**인덱스**:
- `departments_pkey` PK (id)
- `departments_tenant_id_code_key` UNIQUE (tenant_id, code)
- `departments_tenant_id_idx` btree (tenant_id)

**핵심 사항**:
- `parent_id` 컬럼 **없음** → **플랫 구조 확정** (5/4 design.md 가정 일치)
- `code`로 부서 식별, `name`은 표시용
- `(tenant_id, code)` UNIQUE → 워크스페이스별 코드 중복 방지

## 2. `spx_department_members` (사용자 ↔ 부서 매핑)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| id | uuid | NOT NULL | | PK |
| tenant_id | uuid | NOT NULL | | |
| department_id | uuid | NOT NULL | | FK 의도 → spx_departments.id |
| account_id | uuid | NOT NULL | | FK 의도 → spx_accounts.id (Dify 기본 'accounts' 5/18 권대리님 `f88f4a6` rename 직결) |
| is_active | boolean | NOT NULL | true | |
| assigned_at | timestamp | NOT NULL | CURRENT_TIMESTAMP | |
| updated_at | timestamp | NOT NULL | CURRENT_TIMESTAMP | |

**인덱스**:
- `department_members_pkey` PK (id)
- `department_members_department_id_idx` btree (department_id)
- `department_members_tenant_id_account_id_key` UNIQUE (tenant_id, account_id)

**핵심 사항**:
- `account_id` = Dify `spx_accounts.id`(5/18 rename 전 `accounts.id`) 직접 매핑 → **H-DASH-14 (sp_users vs accounts 매핑 불일치) 자동 회피** ✨
- `(tenant_id, account_id)` UNIQUE → **한 사용자는 한 부서에만** (다중 부서 X). 5/4 spec의 `sp_user_departments` 다대다 가정과 다름 — 단일 부서 모델로 단순화됨
- 사용자 → 부서 조회: `spx_accounts JOIN spx_department_members ON account_id = spx_accounts.id JOIN spx_departments ON id = department_id`

## 3. `spx_resource_ownership` (객체 ↔ 부서/사용자 소유) ⭐ 우리 대시보드 핵심

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| id | uuid | NOT NULL | | PK |
| tenant_id | uuid | NOT NULL | | |
| resource_type | varchar(50) | NOT NULL | | 'app' / 'dataset' / 'tool' (운영 시 검증) |
| resource_id | uuid | NOT NULL | | apps.id / datasets.id / tools.id 매핑 |
| owner_department_id | uuid | **NULL** | | NULL = 부서 미배정 → H-DASH-04 fallback 패턴 자연 작동 |
| owner_account_id | uuid | NULL | | 사용자 직접 소유 (부서 소유 아닌 경우) |
| visibility_scope | varchar(50) | NOT NULL | | 가시성 정책 ('private'/'department'/'tenant'/'public' 등 — 운영 시 검증) |
| created_by | uuid | NOT NULL | | 생성자 account_id |
| created_at | timestamp | NOT NULL | CURRENT_TIMESTAMP | |
| updated_at | timestamp | NOT NULL | CURRENT_TIMESTAMP | |

**인덱스**:
- `resource_ownership_pkey` PK (id)
- `resource_ownership_owner_department_id_idx` btree (owner_department_id)
- `resource_ownership_tenant_id_resource_type_resource_id_key` UNIQUE (tenant_id, resource_type, resource_id)

**핵심 사항**:
- 5/4 spec의 `sp_object_ownership` 역할 — 컬럼명 + 보너스 컬럼 추가만 다름
- `owner_department_id` **nullable** = "미배정" 자연 표현 → H-DASH-04 회피
- `Dify 무수정 원칙` 지킴 — `apps`/`datasets`/`tools` 테이블에 FK 추가 안 됨, 별도 매핑 테이블만
- `(tenant_id, resource_type, resource_id)` UNIQUE → 한 객체는 한 소유 정책

## 4. `spx_resource_permissions` (RBAC 권한 — 우리 대시보드 무관)

| 컬럼 | 타입 | NULL | 기본값 | 설명 |
|------|------|------|--------|------|
| id | uuid | NOT NULL | | |
| tenant_id | uuid | NOT NULL | | |
| resource_type | varchar(50) | NOT NULL | | |
| resource_id | uuid | NOT NULL | | |
| principal_type | varchar(50) | NOT NULL | | 권한 받는 주체 ('account'/'department'/'role'?) |
| principal_id | uuid | NOT NULL | | |
| action | varchar(50) | NOT NULL | | 액션 ('view'/'edit'/'delete' 등) |
| effect | varchar(20) | NOT NULL | 'allow' | 'allow' / 'deny' |
| granted_by | uuid | NOT NULL | | 권한 부여자 |
| created_at | timestamp | NOT NULL | CURRENT_TIMESTAMP | |
| expires_at | timestamp | NULL | | 권한 만료 (영구면 NULL) |

**인덱스**:
- `resource_permissions_pkey` PK (id)
- `resource_permissions_principal_idx` btree (tenant_id, principal_type, principal_id)
- `resource_permissions_resource_idx` btree (tenant_id, resource_type, resource_id)
- `resource_permissions_unique_entry_key` UNIQUE (tenant_id, resource_type, resource_id, principal_type, principal_id, action)

**우리 대시보드 무관**: 권한 검증용. admin 권한자만 대시보드 보니 그 자체가 권한 게이트. 집계 시 권한 필터 거치지 않음.

## 5. `spx_rbac_audit_logs` (RBAC 변경 감사 — 우리 대시보드 무관)

| 컬럼 | 타입 | NULL | 설명 |
|------|------|------|------|
| id | uuid | NOT NULL | |
| tenant_id | uuid | NOT NULL | |
| actor_account_id | uuid | NOT NULL | 변경한 사용자 |
| action | varchar(100) | NOT NULL | 'department.create' 등 |
| resource_type | varchar(50) | NULL | |
| resource_id | uuid | NULL | |
| target_principal_type | varchar(50) | NULL | |
| target_principal_id | uuid | NULL | |
| before_json | json | NULL | 변경 전 상태 |
| after_json | json | NULL | 변경 후 상태 |
| created_at | timestamp | NOT NULL | CURRENT_TIMESTAMP |

**인덱스**:
- `rbac_audit_logs_pkey` PK
- `rbac_audit_logs_resource_idx` btree (tenant_id, resource_type, resource_id)
- `rbac_audit_logs_tenant_id_idx` btree (tenant_id, created_at)

**우리 대시보드 무관**: RBAC 변경 추적. 승랑님의 `audit.audit_events` (다른 영역)와 별개.

## 대시보드 쿼리 패턴 (정정 후)

### 부서별 오브젝트 수 (dept-cumulative)

```sql
SELECT
  COALESCE(d.name, '미배정') AS department_name,
  o.resource_type,
  COUNT(*) AS cnt
FROM apps a
LEFT JOIN spx_resource_ownership o
  ON o.resource_id = a.id
 AND o.resource_type = 'app'
 AND o.tenant_id = a.tenant_id
LEFT JOIN spx_departments d
  ON d.id = o.owner_department_id
WHERE a.tenant_id = :tenant_id
GROUP BY d.name, o.resource_type;
```

> `LEFT JOIN` + `COALESCE` = H-DASH-04 미배정 fallback. `owner_department_id NULL`이면 자동 "미배정" 처리.

### 미배정 앱 카운트 (검증용)

```sql
SELECT COUNT(*) AS unassigned
FROM apps a
WHERE a.tenant_id = :tenant_id
  AND NOT EXISTS (
    SELECT 1 FROM spx_resource_ownership o
    WHERE o.resource_type = 'app'
      AND o.resource_id = a.id
      AND o.tenant_id = a.tenant_id
      AND o.owner_department_id IS NOT NULL
  );
```

### 부서별 활성 사용자 (dept-dau)

```sql
-- messages 기반 (account_id → spx_department_members → spx_departments)
SELECT
  COALESCE(d.name, '미배정') AS department_name,
  COUNT(DISTINCT m.from_end_user_id) AS dau
FROM messages m
LEFT JOIN spx_department_members dm
  ON dm.account_id = m.from_end_user_id
 AND dm.tenant_id = m.tenant_id
 AND dm.is_active = true
LEFT JOIN spx_departments d
  ON d.id = dm.department_id
WHERE m.created_at >= NOW() - INTERVAL '24 hours'
  AND m.invoke_from != 'debugger'
  AND m.from_end_user_id IS NOT NULL
  AND m.tenant_id = :tenant_id
GROUP BY d.name;
```

> `is_active = true` 필터 = 비활성 멤버 제외.

### 부서별 호출 수 (dept-call-count)

```sql
SELECT
  COALESCE(d.name, '미배정') AS department_name,
  COUNT(*) AS call_count
FROM messages m
LEFT JOIN spx_resource_ownership o
  ON o.resource_id = m.app_id
 AND o.resource_type = 'app'
 AND o.tenant_id = m.tenant_id
LEFT JOIN spx_departments d
  ON d.id = o.owner_department_id
WHERE m.created_at BETWEEN :start AND :end
  AND m.invoke_from != 'debugger'
  AND m.tenant_id = :tenant_id
GROUP BY d.name;
```

> H-DASH-01 AppMode 분기 + H-DASH-03 디버깅 필터 적용된 형태. WORKFLOW는 별도 `workflow_runs` 쿼리.

## 5/4 sp_ 가정과의 매핑 표 (정정 가이드)

| 5/4 가정 (sp_) | 실제 (회사 명명, `spx_` 접두사) | 매핑 |
|---|---|---|
| `sp_users` | `spx_accounts` (Dify 기본 `accounts` 5/18 rename) + `spx_department_members` (부서 매핑만) | account_id 직접 사용 |
| `sp_departments` | `spx_departments` | plat 구조 동일 (접두사는 5/19 `spx_`로 확정) |
| `sp_user_departments` | `spx_department_members` | 다대다 → 다대일 단순화 |
| `sp_object_ownership` | `spx_resource_ownership` | resource_type/_id, owner_department_id, +visibility_scope |
| `sp_object_permissions` | `spx_resource_permissions` | 우리 대시보드 무관 |
| (없음) | `spx_rbac_audit_logs` | RBAC 변경 감사 (우리 대시보드 무관) |

## 컬럼 이름 갱신 표 (sp_ → 실제)

| 5/4 컬럼 | 실제 컬럼 | 비고 |
|---|---|---|
| `sp_object_ownership.object_type` | `spx_resource_ownership.resource_type` | varchar(50), 'app'/'dataset'/'tool' |
| `sp_object_ownership.object_id` | `spx_resource_ownership.resource_id` | uuid |
| `sp_object_ownership.department_id` | `spx_resource_ownership.owner_department_id` | nullable |
| `sp_object_ownership.owner_user_id` | `spx_resource_ownership.owner_account_id` | nullable, account_id 직접 |
| (없음) | `spx_resource_ownership.visibility_scope` | 가시성 정책, 보너스 |
| `sp_user_departments.user_id` | `spx_department_members.account_id` | account_id 직접 |
| `sp_user_departments.department_id` | `spx_department_members.department_id` | 동일 |
| `sp_users.id` | `spx_accounts.id` (Dify 기본 `accounts` 5/18 rename) | 별도 sp_users 없음 |

## 데이터 상태 (2026-05-12 실데이터 관찰)

> 5/6 시점: 0건. 5/12 재조사 시 mock fixture(`rbac_mock.sql` + `oltp_mock.sql`) 적용 후 데이터 존재 확인.
> 검증 명령: `docker compose exec db_postgres psql -U postgres -d dify -c "SELECT ... FROM <table>"`

### 테이블별 행 수

| 테이블 | 행 수 | 비고 |
|--------|------:|------|
| departments | 5 | IT/MKT/FIN/HR/EXT |
| department_members | 26 | accounts 매핑 25/26 (1건 mock UUID 불일치) |
| resource_ownership | 24 | app:11, tool:8, dataset:5 |
| resource_permissions | 0 | 대시보드 무관 |
| rbac_audit_logs | 0 | 대시보드 무관 |

### resource_ownership 상세 분포

| 지표 | 값 | 비고 |
|------|---:|------|
| resource_type: app | 11 | Dify apps 전체 등록됨 (누락 0) |
| resource_type: tool | 8 | |
| resource_type: dataset | 5 | Dify datasets 전체 등록됨 (누락 0) |
| owner_department_id NULL | 4/24 (16.7%) | 미배정 → H-DASH-04 fallback |
| owner_account_id NULL | 5/24 (20.8%) | 부서 소유 1건 + 완전미배정 4건 |
| created_by NULL | 0/24 (0%) | NOT NULL 제약 작동 |
| visibility_scope: department | 20 (83.3%) | |
| visibility_scope: private | 4 (16.7%) | 미배정 리소스와 일치 |
| created_at 분포 | 전체 2026-05 | mock 일괄 INSERT |

### accounts 매핑률 (top-owners 핵심)

| 매핑 | matched | unmatched | 비고 |
|------|--------:|----------:|------|
| created_by → accounts | 7/24 | 17/24 | **17건은 mock fixture `549120df-...` 단일 UUID** (실 accounts에 없음). 운영 시 RBAC UI가 `accounts.id`로 채우므로 문제 없음 |
| owner_account_id → accounts | 18/19 | 1/19 | 같은 mock UUID 1건. **top-owners는 owner_account_id 기준 사용 권장** |
| department_members.account_id → accounts | 25/26 | 1/26 | 같은 mock UUID |

### departments 분포

| code | name | members | 비고 |
|------|------|--------:|------|
| IT | IT 본부 | 8 | 최대 부서 |
| MKT | 마케팅 | 6 | |
| FIN | 재무 | 5 | |
| HR | 인사 | 4 | |
| EXT | 외부 | 3 | 외부 사용자 버킷 |

- **플랫 구조 확정**: `parent_id` 컬럼 없음 (SELECT 시 에러)
- **다중 부서 0건**: `(tenant_id, account_id)` UNIQUE 제약 + 실제 0건 확인

### Mock vs 운영 차이 (H-DASH-17 방어)

| 항목 | Mock (현재) | 운영 (예상) |
|------|------------|------------|
| created_by 매핑률 | 29% (mock UUID 문제) | ~100% (RBAC UI가 세션 account_id 사용) |
| 레거시 앱 미등록 | 0건 (mock 완전 등록) | **다수** — RBAC 도입 전 앱은 resource_ownership에 행 없음 → H-DASH-04 필수 |
| resource_type | 3종 확인 (app/dataset/tool) | 동일 3종 예상 |
| visibility_scope | department/private 2종 | tenant/public 추가 가능 |

> **운영 최대 리스크**: 레거시 앱의 spx_resource_ownership 미등록. Mock에서는 0건이라 보이지 않지만, 운영에서는 **apps LEFT JOIN spx_resource_ownership** 패턴 + COALESCE 미배정 fallback 필수.

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] — H-DASH-04 (미배정 fallback), H-DASH-13 (RBAC 스키마 변경 영향), H-DASH-14 (users vs accounts), H-DASH-17 (관찰 부족 함정 — 본 정정의 발견 트리거)
- [[3. 프로젝트/spx-agent/hdd/specs/design/dept-objects.md]] — 부서별 오브젝트 SQL (정정 필요)
- [[3. 프로젝트/spx-agent/hdd/specs/design/dept-activity.md]] — 부서별 활동 표 SQL (정정 필요)
- [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md]] — KPI 카드 SQL (총 오브젝트 부분 정정 필요)
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] — 5/6 변경 이력
- [[0. Inbox/Phase2 RBAC 테이블 명세 캡처 (feat-rbac)]] — 본 정정의 1차 캡처 (정정 완료 후 삭제 가능)
- [[0. Inbox/Phase2 KPI 드릴스루 데이터 소스 매트릭스]] — 12종 컴포넌트 매트릭스 (정정 필요)
