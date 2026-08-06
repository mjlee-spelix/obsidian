확인 완료. 슬라이드 28의 데이터 모델은 **5개 RDB 테이블**입니다:
1. `users` (id PK, sub UNIQUE, email, name, role)
2. `departments`
3. `user_departments`
4. `object_ownership`
5. `object_permissions`

(Redis는 RDB가 아니라 캐시이므로 DDL 대상에서 제외)

PostgreSQL 15-alpine 기준으로 DDL을 작성하겠습니다.

---

```sql
-- =====================================================================
-- Keycloak 인증 + 로컬 RBAC 데이터 모델
-- DBMS: PostgreSQL 15 (postgres:15-alpine)
-- 슬라이드 28 데이터 저장 전략에 따라 구성
-- =====================================================================

-- 확장 기능: UUID 생성기. 15-alpine 기본 이미지에 포함되어 있음
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ENUM 타입 정의 (CHECK 제약 대신 ENUM으로 명시)
CREATE TYPE user_role        AS ENUM ('admin', 'user', 'guest');
CREATE TYPE object_type      AS ENUM ('app', 'dataset', 'tool');
CREATE TYPE grantee_type     AS ENUM ('user', 'department');
CREATE TYPE permission_level AS ENUM ('viewer', 'editor');


-- =====================================================================
-- 1) users
--    - id     : 로컬 UUID (외래키 대상, 안정적)
--    - sub    : OIDC subject (IdP 매핑 키, 동일성 확인용)
--    - role   : 시스템 역할 (오브젝트 권한이 아닌, 앱 전체 역할)
--    - JIT 프로비저닝: 첫 로그인 시 INSERT, 이후 로그인 시 email/name UPDATE
-- =====================================================================
CREATE TABLE users (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    sub         VARCHAR(255) NOT NULL UNIQUE,                      -- OIDC subject claim
    email       VARCHAR(320) NOT NULL,                             -- RFC 5321 max
    name        VARCHAR(255) NOT NULL,
    role        user_role    NOT NULL DEFAULT 'user',
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email     ON users (LOWER(email));
CREATE INDEX idx_users_role      ON users (role) WHERE is_active = TRUE;


-- =====================================================================
-- 2) departments
--    - Keycloak group claim 누적으로 자동 생성됨
--    - 관리자가 수동으로 보완 가능 (이름 변경, 비활성화)
--    - parent_id 로 트리 구조 표현 (전사 → IT본부 → 개발팀)
-- =====================================================================
CREATE TABLE departments (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    code        VARCHAR(255) NOT NULL UNIQUE,                      -- Keycloak group path와 매칭 (예: "/IT/dev")
    name        VARCHAR(255) NOT NULL,
    parent_id   UUID         REFERENCES departments(id) ON DELETE SET NULL,
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_dept_no_self_parent CHECK (parent_id IS NULL OR parent_id <> id)
);

CREATE INDEX idx_departments_parent ON departments (parent_id);
CREATE INDEX idx_departments_active ON departments (is_active) WHERE is_active = TRUE;


-- =====================================================================
-- 3) user_departments
--    - 다대다: 한 사용자가 여러 부서에 동시 소속 가능
--    - 매 로그인마다 JWT groups claim으로 갱신 (없으면 INSERT, 있으면 UPDATE last_seen_at)
-- =====================================================================
CREATE TABLE user_departments (
    user_id        UUID        NOT NULL REFERENCES users(id)        ON DELETE CASCADE,
    department_id  UUID        NOT NULL REFERENCES departments(id)  ON DELETE CASCADE,
    assigned_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),               -- 처음 매핑된 시각
    last_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),               -- 가장 최근 JWT claim 확인 시각
    PRIMARY KEY (user_id, department_id)
);

-- 부서 → 사용자 조회용 (부서별 멤버 목록 화면)
CREATE INDEX idx_user_departments_dept ON user_departments (department_id);


-- =====================================================================
-- 4) object_ownership
--    - 모든 보호 대상 오브젝트(App, Knowledge Base, Custom Tool)의 소유권
--    - object_type ∈ {app, dataset, tool}, object_id는 Dify 본체 테이블의 PK 참조
--      ※ Dify 본체에는 외래키를 걸지 않음 (Dify 무수정 원칙)
--    - owner_user_id 는 한번 정해지면 변경 불가 (created_by 의미)
--    - department_id 는 생성 당시 owner의 소속 부서 중 선택된 것 (이관 가능)
-- =====================================================================
CREATE TABLE object_ownership (
    object_type    object_type NOT NULL,
    object_id      UUID        NOT NULL,                            -- Dify의 apps.id / datasets.id / 도구 id
    owner_user_id  UUID        NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    department_id  UUID                 REFERENCES departments(id) ON DELETE SET NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (object_type, object_id)
);

CREATE INDEX idx_ownership_owner  ON object_ownership (owner_user_id);
CREATE INDEX idx_ownership_dept   ON object_ownership (department_id);

-- owner_user_id 변경 불가 (UPDATE 트리거)
CREATE OR REPLACE FUNCTION trg_object_ownership_owner_immutable()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF NEW.owner_user_id IS DISTINCT FROM OLD.owner_user_id THEN
        RAISE EXCEPTION 'object_ownership.owner_user_id is immutable (object_type=%, object_id=%)',
            OLD.object_type, OLD.object_id;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER object_ownership_owner_immutable
    BEFORE UPDATE ON object_ownership
    FOR EACH ROW EXECUTE FUNCTION trg_object_ownership_owner_immutable();


-- =====================================================================
-- 5) object_permissions
--    - 부여된 viewer/editor 권한
--    - grantee_type ∈ {user, department}, grantee_id는 type에 따라 users/departments 참조
--      (다형 외래키이므로 DB 레벨 FK 대신 CHECK + 애플리케이션 레벨 보장)
--    - 동일 (object, grantee) 중복 방지: PRIMARY KEY로 강제
-- =====================================================================
CREATE TABLE object_permissions (
    object_type    object_type        NOT NULL,
    object_id      UUID               NOT NULL,
    grantee_type   grantee_type       NOT NULL,
    grantee_id     UUID               NOT NULL,
    level          permission_level   NOT NULL,
    granted_by     UUID               NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    granted_at     TIMESTAMPTZ        NOT NULL DEFAULT NOW(),
    PRIMARY KEY (object_type, object_id, grantee_type, grantee_id),
    FOREIGN KEY (object_type, object_id)
        REFERENCES object_ownership (object_type, object_id) ON DELETE CASCADE
);

-- 권한 평가 시 사용자별·부서별 권한을 빠르게 합집합 조회
CREATE INDEX idx_obj_perm_user_grantee
    ON object_permissions (grantee_type, grantee_id);

CREATE INDEX idx_obj_perm_object
    ON object_permissions (object_type, object_id);

-- grantee_type별 grantee_id 정합성 검증 (FK 대용)
CREATE OR REPLACE FUNCTION trg_object_permissions_grantee_check()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF NEW.grantee_type = 'user' THEN
        IF NOT EXISTS (SELECT 1 FROM users WHERE id = NEW.grantee_id) THEN
            RAISE EXCEPTION 'grantee_id % does not exist in users', NEW.grantee_id;
        END IF;
    ELSIF NEW.grantee_type = 'department' THEN
        IF NOT EXISTS (SELECT 1 FROM departments WHERE id = NEW.grantee_id) THEN
            RAISE EXCEPTION 'grantee_id % does not exist in departments', NEW.grantee_id;
        END IF;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER object_permissions_grantee_check
    BEFORE INSERT OR UPDATE ON object_permissions
    FOR EACH ROW EXECUTE FUNCTION trg_object_permissions_grantee_check();


-- =====================================================================
-- updated_at 자동 갱신 트리거 (공통)
-- =====================================================================
CREATE OR REPLACE FUNCTION trg_set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

CREATE TRIGGER users_set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION trg_set_updated_at();

CREATE TRIGGER departments_set_updated_at
    BEFORE UPDATE ON departments
    FOR EACH ROW EXECUTE FUNCTION trg_set_updated_at();

CREATE TRIGGER object_ownership_set_updated_at
    BEFORE UPDATE ON object_ownership
    FOR EACH ROW EXECUTE FUNCTION trg_set_updated_at();
```

---

## 설계 노트

### 1. UUID 생성: `pgcrypto.gen_random_uuid()`
PostgreSQL 13+ 기본 제공. `uuid-ossp` 대신 `pgcrypto`를 쓰는 게 가볍습니다. 15-alpine 이미지에 contrib로 포함되어 있어 별도 설치 불필요.

### 2. ENUM vs CHECK
ENUM을 선택한 이유:
- 값 추가 시 `ALTER TYPE ... ADD VALUE` 한 줄
- 인덱스에서 4바이트 정수처럼 동작 (저장 효율)
- pgAdmin/DBeaver 등에서 드롭다운으로 보임

단점: 값 **삭제·이름변경**은 PostgreSQL에서 어렵습니다. 만약 향후 값 변경이 잦을 것 같으면 `VARCHAR + CHECK`로 바꿀 수 있습니다.

### 3. `object_ownership.object_id` 외래키 미연결
`object_type='app'`이면 Dify의 `apps` 테이블, `'dataset'`이면 `datasets`, `'tool'`이면 `tool_providers`를 참조해야 하는데, **다형 참조(polymorphic FK)** 는 PostgreSQL이 직접 지원하지 않습니다. 또한 **Dify 본체 무수정 원칙**상 Dify 테이블에 트리거를 걸 수도 없습니다. → 정합성은 애플리케이션 레이어에서 보장하고, 끊어진 참조(orphan)는 야간 배치로 청소.

### 4. `owner_user_id` 변경 불가
슬라이드 28에 명시한 "한번 정해지면 변경 불가" 원칙을 트리거로 강제. 애플리케이션 코드가 실수로 UPDATE해도 DB 레벨에서 막힙니다.

### 5. 권한 평가 인덱스 전략
가장 빈번한 쿼리는:
```sql
-- "이 사용자에게 부여된 모든 권한 + 사용자가 속한 부서들에 부여된 모든 권한"
SELECT * FROM object_permissions 
 WHERE (grantee_type='user'       AND grantee_id=$1)
    OR (grantee_type='department' AND grantee_id IN (SELECT department_id FROM user_departments WHERE user_id=$1));
```
→ `idx_obj_perm_user_grantee` 가 두 조건 모두 커버.

### 6. CASCADE 정책
- 사용자 삭제 → `user_departments` CASCADE (소속 자동 정리)
- 사용자 삭제 → `object_ownership.owner_user_id` **RESTRICT** (생성자 흔적 보존; 삭제 전 이관 필수)
- 사용자 삭제 → `object_permissions.granted_by` **RESTRICT** (감사 추적)
- 오브젝트 ownership 삭제 → `object_permissions` CASCADE (권한 자동 정리)

### 7. 추가 권장
- **백업**: PG15는 논리 백업 `pg_dump --column-inserts` 사용
- **마이그레이션**: 위 DDL은 Flyway/Liquibase의 `V1__init.sql` 1개 파일로 관리 권장
- **Redis 캐시 키 컨벤션**: `perm:user:{user_id}` (TTL 5~15분), `dept:user:{user_id}` (TTL 5분) — 권한 변경 시 명시적 invalidate

---

원하시면:
- **Flyway/Liquibase 형식**으로 변환
- **샘플 데이터 INSERT 문** 추가
- **Dify의 apps.id 같은 외부 PK 타입**(VARCHAR/UUID)에 맞춘 `object_id` 타입 조정
- **Row-Level Security (RLS)** 정책 예시 추가

해드릴 수 있습니다.