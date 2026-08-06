---
title: Dify Community 단일 워크스페이스 부서형 RBAC 상세 설계안
version: 1.1.0
status: draft
owner: ChatGPT
updated_at: 2026-04-20
methodology: HDD
hdd_phase: detailed_design
source_of_truth:
  - 요구사항: 단일 workspace / 부서 기반 리소스 경계 / 오브젝트별 권한 / 사용자 1인 1부서
  - 제약: Dify Community Edition
upstream:
  - requirements/rbac_requirements.md
  - architecture/dify_extension_architecture.md
downstream:
  - implementation/backend_rbac_service.md
  - implementation/frontend_rbac_ui.md
  - implementation/db_migration_plan.md
  - tests/rbac_test_strategy.md
change_rules:
  - 사용자 한 명은 정확히 하나의 부서에만 속한다.
  - 기존 Dify workspace role은 제거하지 않고 상위 게이트로 유지한다.
  - Knowledge(Dataset)는 Dify 기존 permission 모델과 동기화한다.
  - App/Workflow/Chatbot은 커스텀 오브젝트 ACL로 제어한다.
---

# 1. 문서 목적

이 문서는 **Dify Community Edition**을 기반으로, **단일 워크스페이스 내부에서 부서별로 Studio 리소스와 Knowledge 리소스를 통제하기 위한 RBAC 상세 설계안**을 정의한다.

이 설계안은 다음 조건을 전제로 한다.

- Community Edition은 **워크스페이스 1개**만 사용한다.
- 사용자는 **정확히 하나의 부서**에만 속한다.
- 인사, 개발 사업부 등 **부서 단위로 리소스 소유권과 접근권한**을 관리한다.
- 기존 Dify의 `Owner / Admin / Editor / Member / Dataset Operator` 역할 체계는 유지한다.
- 그 위에 **부서 + 오브젝트 단위 ACL 레이어**를 추가한다.

Dify는 기본적으로 워크스페이스 중심 권한 모델을 사용하며, Community Edition은 설치 시 단일 워크스페이스를 생성하고 해당 공간 안에서 협업하도록 설계되어 있다. 또한 팀 역할은 `Owner`, `Admin`, `Editor`, `Member`, `Dataset Operator` 중심으로 제공된다. 따라서 본 설계는 기존 역할 모델을 대체하지 않고, 그 위에 조직형 RBAC를 얹는 구조를 취한다.

---

# 2. 변경된 핵심 전제

기존 초안과 달리, 본 설계는 다음 전제를 명시적으로 채택한다.

## 2.1 사용자 1인 1부서

한 사용자는 오직 **하나의 부서**에만 속할 수 있다.

의미는 다음과 같다.

- `department_members`에서 한 사용자에 대해 다중 소속을 허용하지 않는다.
- 생성 시 기본 소유 부서를 추론할 때 별도 선택 로직이 단순해진다.
- 부서 권한 전개(expansion)가 단순해진다.
- 부서 이동은 “기존 부서 소속 종료 + 새 부서 재할당”의 형태로 처리한다.

이 제약으로 인해 권한 해석이 단순해지고, UI/운영 복잡도가 낮아진다. 반면 겸직/매트릭스 조직 표현은 포기한다.

## 2.2 조직 RBAC는 사용자 직접 권한 + 부서 권한을 모두 지원

한 사용자가 하나의 부서만 갖더라도, 권한 자체는 다음 두 방식으로 부여할 수 있어야 한다.

- **부서 권한**: 해당 부서 전체에 일괄 부여
- **사용자 직접 권한**: 특정 사용자만 예외적으로 허용

예를 들어,

- 인사부 전체는 인사용 챗봇 `view`, `execute`
- 인사부 담당자 2명만 `edit`
- 인사부장만 `manage_permission`

형태를 지원해야 한다.

---

# 3. 목표 범위

본 설계가 다루는 보호 대상은 아래와 같다.

- App
- Workflow
- Chatbot / Agent App
- Knowledge Dataset
- Dataset Document(선택 적용)
- 향후 Tool / Prompt Asset / API Key로 확장 가능

권한 제어 목표는 다음과 같다.

1. 사용자는 자신이 속한 부서 기준으로 리소스를 생성할 수 있어야 한다.
2. 리소스는 특정 부서에 소속되어야 한다.
3. 리소스마다 공개 범위와 세부 액션 권한을 설정할 수 있어야 한다.
4. 기존 Dify 기본 역할과 충돌 없이 동작해야 한다.
5. Knowledge는 Dify의 기존 Dataset permission 모델과 정합성을 유지해야 한다.

---

# 4. 설계 원칙

## 4.1 기존 Dify role은 상위 게이트로 유지

Dify는 팀 역할에 따라 가능한 작업 범위를 이미 구분한다. 예를 들어 문서상 `Editor`는 앱과 지식 베이스를 관리할 수 있고, `Dataset Operator`는 데이터셋 중심 작업을 수행한다. 따라서 커스텀 RBAC는 이 모델을 무시하면 안 되고, 반드시 **기본 role과 교집합**으로 동작해야 한다.

최종 허용 원칙은 다음과 같다.

```text
허용 = Dify 기본 role capability 허용 AND 커스텀 조직 RBAC 허용
```

예시:

- `Member`에게 커스텀으로 `workflow:create`를 줘도 생성 불가
- `Editor`라도 해당 Workflow에 `edit` 권한이 없으면 수정 불가
- `Admin` override는 정책으로 별도 결정

## 4.2 부서는 워크스페이스 내부의 논리 경계

Community Edition은 단일 워크스페이스 구조이므로, 부서는 별도 workspace가 아니라 **논리적 경계**로 표현한다. 즉, “인사 워크스페이스”를 만드는 것이 아니라, 하나의 workspace 안에서 “인사부 소유 리소스”와 “개발부 소유 리소스”를 구분한다.

## 4.3 모든 리소스는 소유 부서를 가진다

보호 대상 리소스는 반드시 하나의 소유 부서를 가져야 한다.

- 생성 시 `owner_department_id`를 저장한다.
- 권한 기본값은 소유 부서를 기준으로 전개한다.
- 부서 이동이 필요한 경우, `transfer` 권한으로만 수행한다.

## 4.4 서버 측 권한 검사를 기본으로 한다

UI 숨김만으로는 권한 보호가 충분하지 않다. 목록, 상세, 수정, 삭제, 배포, 실행 API 모두에서 서버 측 권한 검사 로직을 수행해야 한다.

---

# 5. 권한 모델

## 5.1 보호 액션 정의

리소스별 세부 권한은 최소 아래 액션으로 분리한다.

- `view` : 목록/상세 조회
- `create` : 새 리소스 생성
- `edit` : 설정, 내용, 플로우 수정
- `delete` : 삭제
- `execute` : 실행, 사용
- `publish` : 배포, 공개 상태 변경
- `manage_permission` : ACL 변경
- `transfer` : 소유 부서/소유자 변경

## 5.2 공개 범위(Visibility Scope)

모든 리소스는 아래 공개 범위를 가진다.

- `private` : 생성자 또는 명시적 ACL만 접근 가능
- `department` : 소유 부서 전체 접근 가능
- `custom` : 사용자/부서 ACL 조합
- `workspace` : 워크스페이스 전체 접근 가능

Knowledge Dataset은 Dify의 기존 permission 모델(`only_me / all_team_members / partial_members`)과 매핑한다. Dify 코드와 UI에는 dataset permission 및 partial member 개념이 이미 존재한다.

## 5.3 권한 주체(Principal)

권한은 아래 두 주체에 부여할 수 있다.

- `user`
- `department`

사용자는 하나의 부서만 가지지만, 특정 개인에게 예외 권한을 줄 수 있어야 하므로 `user` principal을 유지한다.

---

# 6. 전체 구조

```text
[Dify 기본 Role]
  Owner / Admin / Editor / Member / Dataset Operator
        ↓ 상위 capability gate
[조직 모델]
  Department (사용자 1인 1부서)
        ↓
[오브젝트 ACL]
  resource ownership + visibility + permission entries
        ↓
[API 권한 판정]
  목록 / 상세 / 생성 / 수정 / 삭제 / 배포 / 실행
```

이 구조에서 핵심은 다음이다.

- 기존 Dify role이 “이 행동을 할 자격이 있는가”를 먼저 본다.
- 커스텀 RBAC는 “그 자격이 있는 사람 중 이 리소스에 대해 실제 허용되는가”를 본다.

---

# 7. DB 스키마 상세 설계

기존 Dify 테이블을 대폭 수정하기보다, 가능한 한 **사이드카(sidecar) 테이블**로 추가한다. 이는 업스트림 Dify 스키마 변경과의 충돌을 줄이고, 하네스 기반 영향 분석 시 변경 범위를 명확히 분리하는 데 유리하다.

## 7.1 departments

```sql
CREATE TABLE departments (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    code VARCHAR(100) NOT NULL,
    name VARCHAR(255) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_by UUID NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (tenant_id, code)
);
```

설명:

- `tenant_id`는 Dify workspace tenant와 연결된다.
- Community Edition 단일 워크스페이스 기준이므로, tenant는 사실상 하나지만 향후 확장성을 위해 유지한다.

## 7.2 department_members

사용자 1인 1부서를 강제하기 위해, `account_id`에 대해 tenant 기준 unique 제약을 둔다.

```sql
CREATE TABLE department_members (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    department_id UUID NOT NULL,
    account_id UUID NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    assigned_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (tenant_id, account_id),
    FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

설명:

- 한 사용자는 같은 tenant 내에서 오직 하나의 부서만 가질 수 있다.
- 부서 이동은 update로 처리하거나, 이력 보존이 필요하면 history 테이블을 별도 둔다.

## 7.3 resource_ownership

리소스의 부서 소유권과 공개 범위를 표준화한다.

```sql
CREATE TABLE resource_ownership (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID NOT NULL,
    owner_department_id UUID NOT NULL,
    owner_account_id UUID NULL,
    visibility_scope VARCHAR(50) NOT NULL,
    created_by UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (tenant_id, resource_type, resource_id),
    FOREIGN KEY (owner_department_id) REFERENCES departments(id)
);
```

설명:

- `owner_department_id`는 필수다.
- `owner_account_id`는 생성자 또는 책임자 개념이 필요할 때 사용한다.
- `visibility_scope`는 `private / department / custom / workspace`

## 7.4 resource_permissions

오브젝트 ACL의 본체다.

```sql
CREATE TABLE resource_permissions (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    resource_type VARCHAR(50) NOT NULL,
    resource_id UUID NOT NULL,
    principal_type VARCHAR(50) NOT NULL,
    principal_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    effect VARCHAR(20) NOT NULL DEFAULT 'allow',
    granted_by UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    UNIQUE (
        tenant_id,
        resource_type,
        resource_id,
        principal_type,
        principal_id,
        action
    )
);
```

설명:

- `principal_type`은 `user` 또는 `department`
- 초기 버전은 `effect=allow`만 사용해도 되지만, 예외 차단이 필요하면 `deny` 확장 가능

## 7.5 audit_logs

권한 변경과 민감 작업을 감사하기 위한 로그 테이블이다.

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    actor_account_id UUID NOT NULL,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50) NULL,
    resource_id UUID NULL,
    target_principal_type VARCHAR(50) NULL,
    target_principal_id UUID NULL,
    before_json JSONB NULL,
    after_json JSONB NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

# 8. 권한 판정 로직 상세

권한 판정은 중앙 서비스에서 일관되게 처리한다.

## 8.1 판정 순서

### Step 1. 사용자와 tenant 확인

- 현재 로그인 사용자가 tenant에 속하는지 확인
- department_members에서 사용자의 단일 부서를 조회

### Step 2. Dify 기본 role capability 확인

예시:

- App/Workflow 생성: `Owner/Admin/Editor`
- Dataset 생성: `Owner/Admin/Editor/Dataset Operator`
- 단순 사용: `Member` 포함 가능

이 capability는 Dify 문서와 기존 코드의 역할 체계를 기준으로 유지한다.

### Step 3. 커스텀 RBAC 확인

확인 대상:

- 생성자인가
- `resource_ownership.visibility_scope`에 의해 허용되는가
- 사용자 직접 ACL이 있는가
- 사용자의 부서에 ACL이 있는가

### Step 4. 최종 허용/거부

최종식:

```text
allow = base_role_allow AND object_acl_allow
```

## 8.2 기본 권한 계산 규칙

### private

- 생성자 기본 `view/edit/delete/publish/manage_permission/execute`
- 명시 ACL 대상만 접근 가능

### department

- 소유 부서 전체 `view/execute`
- 생성자 `edit/delete/publish/manage_permission`
- 필요 시 템플릿에 따라 부서 전체 `edit` 부여 가능

### custom

- 사용자/부서 ACL에만 의존
- 생성자는 기본 관리자 권한 보유

### workspace

- 워크스페이스 전체 `view/execute`
- 수정 관련 권한은 명시 ACL 또는 생성자/운영 담당자만

---

# 9. 리소스별 정책

## 9.1 App / Workflow / Chatbot

Dify는 Dataset처럼 세밀한 앱 단위 권한 모델을 기본 제공하지 않는다. 따라서 이 리소스군은 커스텀 ACL의 핵심 적용 대상이다. 관련 기능 요청 이슈들에서도 앱 단위 custom group 관리 필요성이 제기된 바 있다.

### 생성 정책

- 사용자는 자신의 단일 소속 부서 기준으로만 생성 가능
- 별도 부서 선택은 일반 사용자에게 허용하지 않음
- `Admin`만 타 부서 소유로 생성 가능

### 기본 ACL 자동 부여

생성 직후 자동 부여:

- 생성자(user): `view/edit/delete/publish/manage_permission/execute/transfer`
- 소유 부서(department): `view/execute`

### 수정/삭제/배포

- 수정: `edit`
- 삭제: `delete`
- 배포: `publish`
- 권한 변경: `manage_permission`
- 소유 부서 변경: `transfer`

## 9.2 Knowledge Dataset

Dataset은 Dify 내부에 이미 `only_me`, `all_team_members`, `partial_members`와 partial member 목록 관리가 존재한다. 따라서 Dataset은 커스텀 ACL을 자체적으로만 적용하지 않고, **기존 Dify permission 모델과 동기화**해야 한다.

### 공개 범위 매핑

- `private` → `only_me`
- `workspace` → `all_team_members`
- `department` → `partial_members`
- `custom` → `partial_members`

### department 처리 방식

사용자가 하나의 부서만 가지므로, `department` 범위는 해당 부서의 전체 계정 목록으로 flatten하여 Dify `partial_members`에 반영한다.

### custom 처리 방식

- 사용자 ACL 직접 지정자는 그대로 추가
- 부서 ACL은 그 부서 사용자들을 펼쳐서 `partial_members`에 반영

### 정합성 유지

Dataset ACL 변경 시 반드시 다음을 수행한다.

1. 커스텀 ACL 계산
2. partial member 대상 계산
3. Dify dataset permission 값 갱신
4. partial member 목록 동기화
5. 감사 로그 기록

---

# 10. API 개입 지점

권한은 UI가 아니라 **백엔드 API**에서 최종 강제되어야 한다.

## 10.1 목록 조회 API

대상:

- App 목록
- Workflow 목록
- Dataset 목록

전략:

- 1차 버전: 기존 목록 조회 후 ACL 필터링
- 성능 최적화 버전: 허용 resource_id 선조회 후 원본 쿼리에 조건 삽입

## 10.2 상세 조회 API

- `view` 권한 없으면 403 또는 404 masking
- 민감 리소스는 404 masking 권장

## 10.3 생성 API

- 사용자 소속 부서 조회
- 해당 부서에 대한 `create` 가능 여부 확인
- 생성 후 `resource_ownership` 생성
- 기본 ACL 자동 부여
- Dataset이면 Dify permission 동기화

## 10.4 수정/삭제/배포 API

- 수정: `edit`
- 삭제: `delete`
- 배포: `publish`
- 권한 편집: `manage_permission`
- 소유부서 변경: `transfer`

## 10.5 실행 API

- 사용 가능 여부는 `execute`로 판정
- 수정 권한과 분리한다

---

# 11. UI/UX 설계

사용자 1인 1부서 전제를 반영하면 UI는 더 단순해진다.

## 11.1 리소스 생성 UI

일반 사용자:

- 부서 선택 UI를 노출하지 않음
- 자동으로 본인 소속 부서가 소유 부서가 됨

관리자:

- 필요 시 부서 선택 허용

공통 입력:

- 공개 범위 선택 (`private / department / custom / workspace`)
- 고급 권한 설정 토글

## 11.2 권한 설정 UI

탭 예시:

- 기본 정보
- 공개 범위
- 사용자 권한
- 부서 권한

예시 표:

| 주체 | 유형 | 권한 |
|---|---|---|
| 인사부 | 부서 | view, execute |
| 김OO | 사용자 | edit |
| 박OO | 사용자 | manage_permission |

## 11.3 목록 UI

리소스 카드/행에 다음 정보 표시:

- 소유 부서
- 공개 범위
- 내 권한 배지

---

# 12. 서비스/모듈 구조 초안

## 12.1 AuthorizationService

```python
class AuthorizationService:
    def can_view(user, resource_type, resource_id) -> bool: ...
    def can_create(user, resource_type, department_id=None) -> bool: ...
    def can_edit(user, resource_type, resource_id) -> bool: ...
    def can_delete(user, resource_type, resource_id) -> bool: ...
    def can_publish(user, resource_type, resource_id) -> bool: ...
    def can_execute(user, resource_type, resource_id) -> bool: ...
    def can_manage_permission(user, resource_type, resource_id) -> bool: ...
    def can_transfer(user, resource_type, resource_id) -> bool: ...
```

내부 체크 순서:

1. tenant 검증
2. 사용자 단일 부서 조회
3. Dify 기본 role capability 확인
4. resource ownership 조회
5. explicit user ACL 조회
6. department ACL 조회
7. visibility scope 반영
8. 정책상 admin override 반영

## 12.2 DatasetAclSyncService

```python
class DatasetAclSyncService:
    def sync_dataset_permission(dataset_id): ...
    def resolve_accounts_for_department(department_id): ...
    def build_partial_member_list(dataset_id): ...
    def validate_dataset_acl_consistency(dataset_id): ...
```

설명:

- 커스텀 ACL을 Dify dataset permission에 맞춰 동기화한다.
- 사용자 1인 1부서 전제 덕분에 department flatten 로직이 단순해진다.

---

# 13. 기본 정책 템플릿

## 13.1 부서 전용 Workflow

- visibility: `department`
- 생성자: `view/edit/delete/publish/manage_permission/execute/transfer`
- 소유 부서: `view/execute`

## 13.2 민감 Dataset

- visibility: `custom`
- 특정 사용자만 `view`
- 담당자만 `edit`
- 부서 전체는 기본 비허용

## 13.3 전사 공용 App

- visibility: `workspace`
- 전체 사용자: `view/execute`
- 운영 담당자만 `edit/publish`

---

# 14. 감사 및 운영 설계

## 14.1 감사 로그 기록 대상

- 리소스 생성
- 공개 범위 변경
- ACL 추가/삭제
- 소유 부서 변경
- Dataset permission 동기화
- 권한 거부 이벤트(선택)

## 14.2 운영자 조회 기능

필수 관리 화면:

- 사용자별 권한 목록
- 리소스별 접근 가능자 목록
- 부서별 소유 리소스 목록
- ACL/Dataset permission 불일치 리소스 목록

---

# 15. 마이그레이션 및 구현 단계

## Phase 1. 조직 모델 구축

- `departments` 테이블 추가
- `department_members` 테이블 추가
- 사용자 1인 1부서 데이터 이관

## Phase 2. 권한 코어 구축

- `resource_ownership`
- `resource_permissions`
- `audit_logs`
- `AuthorizationService`

## Phase 3. App/Workflow 보호

- 생성 시 ownership 기록
- 목록/상세 ACL 필터링
- 수정/삭제/배포 보호

## Phase 4. Dataset 연동

- Dataset ACL 동기화 서비스 구축
- `visibility_scope` ↔ Dify permission 매핑
- partial_members 동기화

## Phase 5. UI 반영

- 공개 범위/권한 설정 UI
- 부서 정보 표시
- 관리자용 감사 조회 화면

## Phase 6. 정합성 검증 자동화

- migration 후 ACL 정합성 검증 스크립트
- Dataset permission sync 검증 배치
- 테스트 자동화

---

# 16. 하네스(HDD) 적용 포인트

이 문서는 HDD(Harness-Driven Development) 관점에서 **상세 설계 문서** 역할을 한다. HDD 관점에서 중요한 것은 문서를 “한 번 쓰고 끝내는 설계서”가 아니라, **변경 전파와 정합성 유지의 기준점**으로 삼는 것이다. 최근 하네스 엔지니어링/HDD 논의에서는 프론트매터 기반 의존성 선언과 변경 영향 분석을 통해 설계-구현-테스트의 정합성을 유지하는 접근이 제시되고 있다.
## 16.1 이 문서의 역할

이 문서는 다음 문서들의 상위 문서로 동작한다.

- backend_rbac_service.md
- frontend_rbac_ui.md
- db_migration_plan.md
- rbac_test_strategy.md

즉, 아래가 변경되면 이 문서부터 먼저 검토해야 한다.

- 사용자-부서 관계 정책
- Dify role 연결 원칙
- Dataset permission 동기화 전략
- 리소스 visibility 규칙

## 16.2 변경 영향 분석 규칙

### 사용자 1인 1부서 정책 변경 시

영향 범위:

- `department_members` 스키마
- 부서 flatten 로직
- 생성 UI/서비스
- 권한 판정 로직
- Dataset partial_members 동기화 로직
- 테스트 시나리오 전체

### Dify dataset permission 정책 변경 시

영향 범위:

- 공개 범위 매핑 규칙
- DatasetAclSyncService
- Dataset 상세/목록 권한 테스트
- 운영 정합성 점검 배치

### Dify 기본 role capability 변경 시

영향 범위:

- AuthorizationService의 base role gate
- App/Workflow/Dataset 생성/수정 API
- 관리자 override 정책

## 16.3 구현 산출물 체크리스트

이 문서를 기준으로 구현 단계에서 반드시 파생되어야 하는 산출물은 다음과 같다.

- [ ] DB migration 문서
- [ ] 권한 서비스 상세 설계 문서
- [ ] Dataset ACL sync 상세 설계 문서
- [ ] 백엔드 API 체크포인트 문서
- [ ] 프론트 권한 설정 UI 문서
- [ ] 테스트 전략 문서
- [ ] 운영 감사/정합성 점검 문서

---

# 17. 오픈 이슈 / 정책 결정 필요 항목

아래 항목은 구현 전 확정되어야 한다.

## 17.1 Admin/Owner override 범위

선택지:

- 전 리소스 완전 override
- 조회만 override
- 인사 등 민감 부서는 별도 제한

## 17.2 부서 이동 시 권한 처리

사용자는 하나의 부서만 가지므로, 부서 이동 시 아래 정책이 필요하다.

- 새 부서 ACL 자동 적용 여부
- 이전 부서 기반 권한 자동 제거 여부
- 사용자 직접 ACL 유지 여부

## 17.3 일반 사용자의 타부서 생성 허용 여부

기본 권장은 **불허**다.

## 17.4 Dataset Document까지 별도 ACL을 둘지 여부

초기 버전은 Dataset 수준 권한만 적용하고, 이후 Document 세분화 필요 시 확장 가능하다.

---

# 18. 결론

본 설계안의 핵심은 다음과 같다.

1. Dify Community의 **단일 workspace 제약**을 유지한다.
2. 사용자는 **정확히 하나의 부서**만 가진다.
3. 모든 보호 리소스는 **하나의 소유 부서**를 가진다.
4. 권한은 **기존 Dify role + 커스텀 오브젝트 ACL의 교집합**으로 판정한다.
5. App/Workflow/Chatbot은 커스텀 ACL로 직접 보호한다.
6. Knowledge Dataset은 **Dify 기존 permission 모델과 동기화**한다.
7. 이 문서는 HDD 관점에서 **상세 설계의 기준 문서**이며, 이후 구현/테스트 문서의 상위 기준점으로 사용한다.

이 설계는 현재 Dify Community가 제공하는 workspace 중심 역할 모델과 Dataset permission 구조를 존중하면서, 단일 workspace 안에서 부서별 조직 운영이 가능하도록 확장하는 현실적인 방안이다.
