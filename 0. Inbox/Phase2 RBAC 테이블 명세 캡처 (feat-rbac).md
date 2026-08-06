---
tags: [프로젝트, dify, AI-Agent, 임시, RBAC]
date: 2026-05-06
status: 임시 캡처 → references/rbac-schema.md 정식 갱신 후 제거
---
# Phase 2 RBAC 테이블 명세 캡처 (feat/rbac 브랜치)

> **캡처 목적**: feat/rbac 브랜치 체크아웃 + `flask db upgrade` 적용 시점에 실 DB에서 직접 추출한 권대리님 RBAC 영역 명세. 원복(KAN-29) 후엔 코드/마이그레이션이 우리 브랜치엔 없어서 다시 못 보니 보관용.
> **다음 단계**: [[3. 프로젝트/spx-agent/references/rbac-schema.md]] 전면 갱신 시 이 명세 참조. 갱신 완료 후 이 임시 노트 삭제.
> **관찰 시점**: 2026-05-06, 본인 로컬 docker 환경 (개발자 분리)

## 핵심 발견 — 명명 표준 차이

5/4 우리 spec은 `sp_` prefix 가정. 권대리님 실제는 **prefix 없음 + 일반화된 명명**.

| 5/4 가정 (sp_) | 실제 (권대리님) | 매핑 |
|---|---|---|
| `sp_departments` | `departments` | ✅ 1:1 (이름만 다름) |
| `sp_users` | `department_members` | ✅ 1:1 (account_id 직접 매핑, H-DASH-14 회피) |
| `sp_object_ownership` | `resource_ownership` | ✅ 1:1 (보너스 컬럼 추가) |
| (없음) | `resource_permissions` | ⚠️ RBAC 핵심, 우리 대시보드 무관 |
| (없음) | `rbac_audit_logs` | ⚠️ RBAC 변경 감사, 우리 대시보드 무관 |

→ **우리 8종 RBAC 의존 컴포넌트 모두 작업 가능**. 김이사님 ETA 문의 불필요.

## 테이블 1 — `departments` (부서 마스터)

```
                             Table "public.departments"
   Column   |            Type             | Collation | Nullable |      Default
------------+-----------------------------+-----------+----------+-------------------
 id         | uuid                        |           | not null |
 tenant_id  | uuid                        |           | not null |
 code       | character varying(100)      |           | not null |
 name       | character varying(255)      |           | not null |
 is_active  | boolean                     |           | not null | true
 created_by | uuid                        |           |          |
 created_at | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 updated_at | timestamp without time zone |           | not null | CURRENT_TIMESTAMP

Indexes:
    "departments_pkey" PRIMARY KEY, btree (id)
    "departments_tenant_id_code_key" UNIQUE CONSTRAINT, btree (tenant_id, code)
    "departments_tenant_id_idx" btree (tenant_id)
```

**데이터**: 0 rows.

**비고**:
- `parent_id` 컬럼 없음 → **플랫 구조 확정** (5/4 design.md 가정 일치)
- `code`로 부서 식별 (예: 'IT', 'MKT', 'FIN'), `name`은 표시용
- `(tenant_id, code)` UNIQUE → 워크스페이스별 코드 중복 방지

## 테이블 2 — `department_members` (사용자 ↔ 부서 매핑)

```
                           Table "public.department_members"
    Column     |            Type             | Collation | Nullable |      Default
---------------+-----------------------------+-----------+----------+-------------------
 id            | uuid                        |           | not null |
 tenant_id     | uuid                        |           | not null |
 department_id | uuid                        |           | not null |
 account_id    | uuid                        |           | not null |
 is_active     | boolean                     |           | not null | true
 assigned_at   | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 updated_at    | timestamp without time zone |           | not null | CURRENT_TIMESTAMP

Indexes:
    "department_members_pkey" PRIMARY KEY, btree (id)
    "department_members_department_id_idx" btree (department_id)
    "department_members_tenant_id_account_id_key" UNIQUE CONSTRAINT, btree (tenant_id, account_id)
```

**데이터**: 0 rows.

**비고**:
- `account_id` = Dify `accounts.id` 직접 매핑 → **H-DASH-14 (sp_users vs accounts 매핑) 자동 회피** ✨
- `(tenant_id, account_id)` UNIQUE → 한 사용자는 한 부서에만 속함 (다중 부서 X)
- 사용자 → 부서 조회: `JOIN department_members ON account_id = accounts.id`

## 테이블 3 — `resource_ownership` (객체 ↔ 부서/사용자 소유) ⭐

```
                              Table "public.resource_ownership"
       Column        |            Type             | Collation | Nullable |      Default
---------------------+-----------------------------+-----------+----------+-------------------
 id                  | uuid                        |           | not null |
 tenant_id           | uuid                        |           | not null |
 resource_type       | character varying(50)       |           | not null |
 resource_id         | uuid                        |           | not null |
 owner_department_id | uuid                        |           |          |
 owner_account_id    | uuid                        |           |          |
 visibility_scope    | character varying(50)       |           | not null |
 created_by          | uuid                        |           | not null |
 created_at          | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 updated_at          | timestamp without time zone |           | not null | CURRENT_TIMESTAMP

Indexes:
    "resource_ownership_pkey" PRIMARY KEY, btree (id)
    "resource_ownership_owner_department_id_idx" btree (owner_department_id)
    "resource_ownership_tenant_id_resource_type_resource_id_key" UNIQUE CONSTRAINT, btree (tenant_id, resource_type, resource_id)
```

**데이터**: 0 rows.

**핵심 컬럼 의미**:
- `resource_type` (varchar(50)): 'app' / 'dataset' / 'tool' (예상값, 실 데이터 받으면 검증)
- `resource_id` (uuid): `apps.id` / `datasets.id` / `tools.id` 매핑
- `owner_department_id` (uuid, **nullable**): NULL = 미배정 → **H-DASH-04 fallback 패턴 자연 작동**
- `owner_account_id` (uuid, nullable): 사용자 직접 소유 (부서 아닌 경우)
- `visibility_scope` (varchar(50)): 가시성 정책 (예상: 'private'/'department'/'tenant'/'public', 실 데이터로 검증)

**우리 대시보드 활용**:
- objects 메트릭 (앱/데이터셋/도구가 어느 부서)
- calls/users/errors 메트릭 (호출/사용 발생한 앱이 어느 부서)
- 표 + 차트 모두 GROUP BY `owner_department_id` 패턴

**JOIN 패턴 예시 (5/4 design.md sp_object_ownership SQL을 정정)**:
```sql
-- 부서별 앱 수
SELECT COALESCE(d.name, '미배정') AS dept, COUNT(*) AS cnt
FROM apps a
LEFT JOIN resource_ownership o
  ON o.resource_id = a.id AND o.resource_type = 'app'
LEFT JOIN departments d
  ON d.id = o.owner_department_id
WHERE a.tenant_id = :tenant_id
GROUP BY d.name;
```

## 테이블 4 — `resource_permissions` (RBAC 권한 — 우리 무관)

```
                               Table "public.resource_permissions"
     Column     |            Type             | Collation | Nullable |          Default
----------------+-----------------------------+-----------+----------+----------------------------
 id             | uuid                        |           | not null |
 tenant_id      | uuid                        |           | not null |
 resource_type  | character varying(50)       |           | not null |
 resource_id    | uuid                        |           | not null |
 principal_type | character varying(50)       |           | not null |
 principal_id   | uuid                        |           | not null |
 action         | character varying(50)       |           | not null |
 effect         | character varying(20)       |           | not null | 'allow'::character varying
 granted_by     | uuid                        |           | not null |
 created_at     | timestamp without time zone |           | not null | CURRENT_TIMESTAMP
 expires_at     | timestamp without time zone |           |          |

Indexes:
    "resource_permissions_pkey" PRIMARY KEY, btree (id)
    "resource_permissions_principal_idx" btree (tenant_id, principal_type, principal_id)
    "resource_permissions_resource_idx" btree (tenant_id, resource_type, resource_id)
    "resource_permissions_unique_entry_key" UNIQUE CONSTRAINT, btree (tenant_id, resource_type, resource_id, principal_type, principal_id, action)
```

**데이터**: 0 rows.

**비고**: 권한 검증용 (RBAC 핵심). 우리 admin 대시보드는 집계만 하므로 **참조 불필요**. 김이사님/태영님 화면용.

## 테이블 5 — `rbac_audit_logs` (RBAC 변경 감사 — 우리 무관)

```
                                 Table "public.rbac_audit_logs"
        Column         |            Type             | Collation | Nullable |      Default
-----------------------+-----------------------------+-----------+----------+-------------------
 id                    | uuid                        |           | not null |
 tenant_id             | uuid                        |           | not null |
 actor_account_id      | uuid                        |           | not null |
 action                | character varying(100)      |           | not null |
 resource_type         | character varying(50)       |           |          |
 resource_id           | uuid                        |           |          |
 target_principal_type | character varying(50)       |           |          |
 target_principal_id   | uuid                        |           |          |
 before_json           | json                        |           |          |
 after_json            | json                        |           |          |
 created_at            | timestamp without time zone |           | not null | CURRENT_TIMESTAMP

Indexes:
    "rbac_audit_logs_pkey" PRIMARY KEY, btree (id)
    "rbac_audit_logs_resource_idx" btree (tenant_id, resource_type, resource_id)
    "rbac_audit_logs_tenant_id_idx" btree (tenant_id, created_at)
```

**데이터**: 0 rows.

**비고**: RBAC 변경(부서 생성/사용자 배정/권한 부여) 감사 로그. 승랑님의 `audit.audit_events`와는 별개. 우리 대시보드 직접 무관.

## Dify 무수정 원칙 검증 ✅

```
docker compose exec db_postgres psql -U postgres -d dify -c "\d apps" | Select-String "department\|dept\|owner"
(빈 결과)
```

→ **`apps` 테이블에 직접 `department_id`/`owner_*` 컬럼 추가 안 됨**. 권대리님이 우리 공통 컨벤션("Dify 기존 테이블 무수정") 지킴. 매핑은 별도 `resource_ownership` 테이블로 분리. 깔끔.

## 갱신 작업 체크리스트 (이 노트 활용해서)

원복 후 처리:

- [ ] **`references/rbac-schema.md` 전면 갱신** — 위 5종 테이블 명세 박기 + 5/4 sp_ 가정 정정 사실 명시
- [ ] **`SESSION_HISTORY.md` 핵심 결정 표** — "회사 RBAC 명명: prefix 없음, `departments`/`department_members`/`resource_ownership`/`resource_permissions`/`rbac_audit_logs`" 박기
- [ ] **`hdd/specs/design/*.md` SQL 5종** — sp_ → 실제 이름 갈아끼우기 (특히 dept-objects, dept-activity, kpi-cards)
- [ ] **`hdd/specs/design/kpi-drill-through.md`** 데이터 소스 매트릭스 갱신 — RBAC 의존 8종 모두 즉시 가능으로 변경
- [ ] **`api/services/admin/*.py` mock 컨트롤러 5종** — sp_ 가정 → 실제 명명 정정 (5/6 commit한 코드)
- [ ] **`hdd/defect-catalog.md`**: H-DASH-13 보강 (회피책 작동 사례) + H-DASH-17 신규 (관찰 부족 함정)
- [ ] **`0. Inbox/Phase2 KPI 드릴스루 데이터 소스 매트릭스.md`** — 매핑 컬럼명 정정 (sp_ → 실제)
- [ ] 갱신 완료 후 **이 임시 노트 삭제**

## 메타 학습 — 추가 학습 후보

> **DB는 git 브랜치와 분리된 영속 상태** — spec 추정보다 `\dt` 한 줄이 정확.
> 신규 도메인 진입 시 spec 작성 전에 **(a) 다른 분 브랜치 `git ls-tree` (b) 우리 환경 docker DB `\dt`** 확인이 의무.
> 5/4 spec 작성 시점에 이걸 안 했기에 sp_ 가정이 5/6까지 잘못 박혀있었음. 회피값 = "RBAC 8종 컴포넌트 김이사님 ETA 대기" 잘못 인식 + 컬럼명 추측 spec 박힘.

→ [[4. 지식노트/]] + `defect-catalog.md` H-DASH-17 후보.

## 관련 노트

- [[3. 프로젝트/spx-agent/references/rbac-schema.md]] — 정식 명세 (이 노트 내용으로 갱신 예정)
- [[3. 프로젝트/spx-agent/references/audit-schema.md]] — 승랑님 audit (별개 영역)
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] — 5/6 변경 이력 추가 예정
- [[0. Inbox/Phase2 KPI 드릴스루 데이터 소스 매트릭스.md]] — sp_ → 실제 매핑 갱신 필요
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] — H-DASH-13 (회피책 작동) + H-DASH-17 (신규)
