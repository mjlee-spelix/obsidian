---
tags: [프로젝트, dify, AI-Agent, keycloak, operations]
type: reference
date: 2026-05-26
last_updated: 2026-05-26
purpose: Keycloak 부서 그룹/사용자 동기화 운영 절차 + 매칭 우선순위 + 트러블슈팅 박제. 5/22 SESSION_HISTORY § Keycloak 부서 그룹 동기화에서 분리.
---

# Keycloak 동기화 운영 절차 (references)

> 5/22 SESSION_HISTORY § Keycloak 부서 그룹 동기화 + § 트러블슈팅에 박힌 사실을 영속 운영 절차 표준으로 분리. SESSION_HISTORY는 시간순 박제라 다음 작업자의 "Keycloak 동기화 어떻게 돌리지?" 검색 hit 곤란 → 본 문서로 추출.
>
> **스크립트 본문은 `.gitignore` 대상** (비밀번호 포함). 본 문서엔 절차 / 입출력 / 매칭 우선순위 / 트러블슈팅만.

## § 1. 배경

- 원본 흐름: **Keycloak 그룹 등록 → DB 반영** (정합성 단방향)
- 5/22 시점 정합성 불일치:
  - `spx_departments` DEPT-01~10 (10개) 박힘 / `keycloak_group_id = NULL` 박힘
  - 목업 사용자 150명 박힘 / Keycloak `user_entity` 미등록
  - → 일괄 동기화 필요
- 본 문서 = 동기화 스크립트의 **반복 운영 절차** + **매칭 우선순위 박제** + **트러블슈팅 표준**
- 스크립트 본문은 `.gitignore` (비밀번호 포함) — 본 문서엔 박지 않음

## § 2. 인스턴스 분리

| 인스턴스 | 주소 | DB | 용도 |
|---|---|---|---|
| **로컬 Keycloak** | `localhost:8180` | `db_postgres:5432/keycloak` (compose 컨테이너) | 개발/테스트 |
| **194 Keycloak** | `192.168.10.194:8080` | 194 서버 자체 DB (로컬과 독립) | 데모/공유 환경 |

> ⚠️ **주의**: 두 인스턴스는 **별도 realm + 별도 user_entity**. 한 쪽에 user 박아도 다른 쪽에 자동 전파 안 됨. 동기화 스크립트는 **대상 인스턴스 환경변수 (`KC_URL` 등)로 분리 실행** 의무.

## § 3. Keycloak 테이블 4종 (실측, 5/22 조사)

| 테이블 | 핵심 컬럼 | 우리 매핑 |
|---|---|---|
| `user_entity` | `id` (UUID, PK) / `username` / `email` / `first_name` / `last_name` / `realm_id` | `id` ↔ `spx_accounts.sub` |
| `user_group_membership` | `group_id` / `user_id` | 사용자 ↔ 부서 그룹 매핑 |
| `credential` | `user_id` / `secret_data` / `credential_data` | 비밀번호 해시 — **직접 SQL 박지 말 것** (admin API 통해서만) |
| `groups` | `id` (UUID, PK) / `name` / `parent_group` / `realm_id` | `id` ↔ `spx_departments.keycloak_group_id`, `name` = DB `code` |

## § 4. 매칭 우선순위 (`api/services/account_service.py:209-365` 기준)

`_sync_department_from_keycloak_groups` 함수가 로그인 시점에 Keycloak JWT claims의 `groups`를 받아 DB `spx_departments`와 매칭:

```
1) UUID 매칭   — spx_departments.keycloak_group_id == kc_group.id → 즉시 매핑 (가장 안전, rename에 영향 없음)
2) code 매칭   — spx_departments.code == path.rsplit("/", 1)[-1] → 매핑 + keycloak_group_id 컬럼에 lazy backfill
3) 신규 생성   — 위 둘 다 실패 시 DepartmentService.create_department(code, name="(미지정)", keycloak_group_id)
                 DepartmentCodeConflictError race 시 1단계로 retry
```

- **Keycloak 그룹명 (`groups.name`) = DB `spx_departments.code` 값** (path leaf, prefix 없음)
- `spx_departments.keycloak_group_id` NULL → 로그인 시 자동 backfill 가능하나, **수동 선반영 권장** (자동 backfill은 매칭 실패 race condition 위험)
- 그룹명 rename 시 코드-동기 블록(파일 L340-355)이 `dept.code`를 leaf로 업데이트 (`try/except IntegrityError`로 충돌 흡수)
- `dept.name`은 신규 생성 후 **절대 덮어쓰지 않음** (사용자 입력 보존)
- `claims["groups"]` 부재/비-list → no-op (멤버십 안전 가드, 빈 list `[]`만 "그룹 없음" → 전체 제거 트리거)

## § 5. 동기화 스크립트 (운영 절차)

> 스크립트 본문은 `.gitignore` 대상. 본 § 는 절차 / 입출력 / 체크리스트만.

### § 5.1 `scripts/sync_dept_keycloak.sh` — 부서 그룹

- **입력**: `spx_departments` 행들 (`code`, `name`)
- **처리**:
  1. 194 Keycloak admin API로 그룹 생성 (`POST /admin/realms/{realm}/groups`)
  2. 응답 `Location` 헤더에서 group_id 추출
  3. DB `spx_departments.keycloak_group_id`에 UPDATE
- **출력**: 그룹 생성 N건 / 스킵(이미 존재) M건 / DB UPDATE 결과
- **사전 조건**:
  - Keycloak admin 계정 토큰 발급 가능
  - 194 DB 직결 환경변수 (`-h 192.168.10.194 -p 5432 -e PGPASSWORD=...`) 준비
- **멱등성**: 이미 존재하는 그룹은 스킵 + 기존 group_id 조회 후 DB backfill만

### § 5.2 `scripts/sync_mock_users_keycloak.py` — 목업 사용자

- **입력**: 목업 사용자 150명 (`username`, `email`, `first_name`, `last_name`, `department_code`, `password`)
- **처리**:
  1. 194 Keycloak admin API로 `user_entity` 생성 (`POST /admin/realms/{realm}/users`)
  2. `credential` 등록 (별도 PUT `reset-password` 엔드포인트 — 직접 SQL 금지)
  3. 부서 `keycloak_group_id` 조회 후 `user_group_membership` 추가 (`PUT /users/{id}/groups/{group_id}`)
  4. 생성된 `user_entity.id`를 `spx_accounts.sub`에 UPDATE
- **출력**: 사용자 생성 N건 / 그룹 배정 N건 / DB sub UPDATE 결과
- **사전 조건**:
  - § 5.1 선행 (부서 그룹 먼저 박혀야 사용자가 그룹에 들어감)
  - `spx_accounts`에 목업 사용자 행 박혀있음 (`username`/`email`로 매칭)
- **멱등성**: `NOT EXISTS` 가드 + 50건마다 토큰 재발급 (H-INFRA-04 방어)

### § 5.3 실행 순서

```
1. scripts/sync_dept_keycloak.sh       (선행 — 부서 그룹)
2. scripts/sync_mock_users_keycloak.py (후행 — 사용자 + 그룹 배정)
```

순서 뒤집으면 사용자 그룹 배정 실패 (그룹 미존재로 `PUT /users/{id}/groups/{group_id}` 404).

## § 6. 트러블슈팅 5종 (5/22 학습 박제)

| # | 증상 | 원인 | 해결 |
|---|---|---|---|
| 1 | 비밀번호 특수문자 (`#$^^`) shell 치환 깨짐 | bash 변수 확장이 `$^` 해석 시도 | **작은따옴표 + `--data-urlencode`** |
| 2 | `docker exec psql`이 로컬 DB로 들어감 | 컨테이너 환경변수 `PGHOST` 기본값 = 로컬 | `-h 192.168.10.194 -p 5432 -e PGPASSWORD=...` 명시로 194 원격 직결 |
| 3 | bash → python 한국어 인자 깨짐 | Windows Git Bash MSYS layer가 인자 cp949 변환 | **전체 python 스크립트로 전환** → 결함 **H-ENV-04** 박제 |
| 4 | 150건 처리 중 401 Unauthorized | Keycloak admin access token TTL 5분 만료 | **50건마다 토큰 재발급** → 결함 **H-INFRA-04** 박제 |
| 5 | `spx_accounts_sub_unique_idx` 충돌 | Keycloak 로그인 자동생성 계정 + 목업 계정 sub 중복 | 자동생성 삭제(예: `김 민준` `3d9faddf-...`) + 목업 계정에 sub 수동 반영 + `NOT EXISTS` 가드 추가 |

## § 7. 관련 결함 (defect-catalog 참조)

- **H-INFRA-04** — Keycloak admin API 토큰 만료 (방어: 50건/4분 단위 재발급, 멱등 보장)
- **H-ENV-04** — Windows Git Bash 멀티바이트 인자 깨짐 (방어: 한글 동반 스크립트는 python 전환)
- **H-DASH-17** (자매) — 외부 의존 명세 추정 함정. Keycloak 실 스키마(§ 3) / 매칭 코드(§ 4)는 5/22 실측 + admin UI 교차 검증으로 자가 검증 완료 (이 가드의 적용 사례)

## § 8. 후속 트리거 — P2 web-ui Keycloak 통합

본 문서는 **P2 (web-ui ↔ Keycloak 연결) 사전 분석의 base 참조**:

- web-ui ↔ Keycloak 통합 시 **이미 박힌 sub / group_id 매핑 재사용** (신규 realm/client 분리 결정 시점에 본 문서 § 3·§ 4 매핑 표 참조)
- 통합 옵션 후보 (사용자 분석 트랙):
  - **SPA-only PKCE** (web-ui 자체 keycloak-js, 백엔드 미통과)
  - **SSR + httpOnly cookie** (Next.js API 경유)
  - **realm 공유 vs web-ui 전용 client 분리** (spx-agent와 같은 realm vs 별도 realm)
- 본 문서 § 4 매칭 우선순위가 **공유 realm 채택 시 그대로 적용** (sub 매핑 단일 진실)
- 분리 realm 채택 시 sub 매핑 별도 설계 + 본 문서의 매칭 우선순위 재사용 가능 여부 별도 검증

상세 분석 트랙은 옵시디언 데일리 노트(`1. Daily/2026-05-2x.md` P2 섹션) 참조.

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|defect-catalog]] — H-INFRA-04 / H-ENV-04 / H-DASH-17
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|rbac-schema]] — `spx_departments` / `spx_accounts` 스키마
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md|SESSION_HISTORY]] — 2026-05-22 § Keycloak 부서 그룹 동기화 (원본 박제)
