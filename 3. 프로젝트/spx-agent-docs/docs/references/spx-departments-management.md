---
title: 사용자/부서 관리 — 운영 흐름 분석
phase: Phase 4 / 클러스터 A / A3
status: 완료 (2026-06-04), 이사님 결정 5·6 반영 (2026-06-04 갱신)
audience: spx-agent 사용자 매뉴얼 작성자 (Phase 4 챕터 집필자)
purpose: |
  A1·A2에서 정립한 권한 모델 코어 위에 워크스페이스 단위 사용자/부서
  운영 흐름만 더한 분석. 부서 CRUD UI·멤버 배정·외부 그룹 동기화
  운영 정책을 박제하고, A1·A2 챕터에서 위임한 "운영자에게 요청" 동선을
  정확한 절차로 묶어준다. **본 문서는 Dify 원본 `team-members-management`
  페이지의 변환 결과 정전**(결정 6) — Phase 4 본격 작성 시 원본 자리에 본
  챕터를 대체 배치한다.
status_history:
  - 2026-06-04 초안: 부서 CRUD UI + KC sync mirror 정책 박제, A1·A2 모호 동선 흡수
  - 2026-06-04 갱신: 결정 5 (KC 추상화 전역 규칙 #6 적용 가이드 §0 신설), 결정 6 (team-members-management 변환 박제)
base_documents:
  - spx-app-permissions-analysis.md   # A1 권한 모델 코어
  - spx-knowledge-permissions.md      # A2 지식 동기화 트리거 미구현 영역
source_documents:
  - dify_rbac_hdd_design.md §15 INV-13, §19.2  # 부서 삭제 거부, 부서 이동 미결
  - RBAC_HANDOVER.md §6, §8.1                  # KC sync, 일상 운영 시나리오
related_decisions:
  - 결정 5 (2026-06-04 회의) — KC 추상화 전역 규칙 #6 신설
  - 결정 6 (2026-06-04 회의) — team-members-management → 사용자·부서 관리 변환
code_verified_at: 2026-06-04
---

# 0. 표기 가이드 — KC 추상화 (전역 규칙 #6)

> 본 절은 분석 산출물·챕터 본문 양쪽 모두에 적용되는 정책. 2026-06-04 결정 5 박제. 본 문서는 이 가이드를 따른다.

## 0.1 표기 정책

| 영역 | 표기 |
|------|------|
| **사용자 매뉴얼 본문**(`ko/use-spx-agent/**/*.mdx`)에서 외부 IdP·인증·관리 시스템을 가리킬 때 | **"외부 시스템(예, Keycloak)"** 또는 **"관리 시스템"** |
| 처음 등장 시 | "외부 시스템(예, Keycloak)" — 예시 1회 표기 |
| 이후 동일 문맥 | "외부 시스템" 또는 "관리 시스템" |
| 작용·동작 설명(mirror 정책 등) | **사실 그대로 유지** — 표기만 추상화, 동작은 손대지 않음 |

## 0.2 적용 사유 (결정 5 출처)

spx-agent 사용 기업마다 IdP가 다를 수 있다. "Keycloak" 직접 표기는 제품 종속 인상을 줘 다른 IdP 사용 환경에서 오해를 부른다. 추상화 표기로 IdP 중립성 유지.

## 0.3 보존 영역 (추상화 예외)

다음 영역은 **Keycloak 직접 표기 유지**:

| 영역 | 사유 |
|------|------|
| 분석 산출물의 코드 식별자(`account_service._sync_department_from_keycloak_groups`, `keycloak_group_id` 컬럼, `KeycloakAdmin` 클래스) | 기술 정확성 — 코드와 1:1 매핑 |
| 분석 산출물의 엔드포인트·테이블·파일 경로 (`api/libs/keycloak_admin.py` 등) | 동일 |
| 메타 정보 (HANDOVER §6 인용, JWT `groups` claim 등) | 인증 메커니즘 식별이 필요한 운영자/개발자 부록 |
| 본 문서의 §5 동작 박제 (mirror 정책 알고리즘) | 분석본은 운영 정전(canonical) — 추상화 X |

> **분석본 vs 챕터 본문 구분**: 분석본(`docs/references/*.md`)은 코드와 직결되는 정전이라 식별자 보존. 챕터 본문(`ko/use-spx-agent/**/*.mdx`)은 사용자 노출이라 추상화. **본 §5 내부도 분석본 영역이라 코드 식별자는 유지**하되, 챕터 작성 시 인용·번역 단계에서 추상화 표기로 옮긴다.

## 0.4 본 문서 안에서의 표기

본 분석본의 본문은 §5(KC 동기화 절)에서 코드 정확도를 위해 "Keycloak" 직접 표기를 유지한다. **단, 챕터 본문 작성 시 사용할 추상화 예시는 §5.6에서 별도 박제** — 작성자가 인용 시 그 예시를 그대로 쓰면 된다.

---

# 1. 본 문서의 위치

본 문서는 [[references/spx-app-permissions-analysis]](A1)과 [[references/spx-knowledge-permissions]](A2)의 권한 모델을 **전제**하고, 워크스페이스 단위 사용자/부서 운영 흐름만 다룬다.

**본 문서는 동시에 두 역할을 한다**:

1. **신규 챕터 "사용자/부서 관리"의 분석 정전** — 부서 CRUD UI·멤버 배정·외부 IdP 그룹 동기화
2. **Dify 원본 `workspace/team-members-management.mdx`의 변환 대체 정전** — 2026-06-04 결정 6에 따라 원본 페이지 자리를 본 챕터로 대체. 단순 삭제가 아니라 **확장 변환**(KC 동기화 + 부서 기능 추가). A4 [[references/spx-workspace-analysis]] §3.3이 본 변환을 인용한다.

A1·A2 챕터에서 "운영자에게 요청"으로 위임된 동선이 본 문서에서 다음과 같이 풀린다:

| A1·A2의 모호한 안내 | A3의 정확한 절차 |
|--------------------|----------------|
| "워크스페이스 운영자에게 강제 재동기화 요청" (A2 FAQ) | §6.2 [[#6.2-부서-멤버-변경-후-지식-검색-권한이-안-맞을-때]] |
| "운영자가 백엔드로 처리" (A1 §6.3 소유 계정 변경) | A4 위임 — Workspace 챕터에서 다룸 |
| "권한 관리자에게 확인 요청" (A1 FAQ DENY 시나리오) | §7.1 [[#7.1-사용자별-권한-목록-운영자-조회]] |

> **인접 분석**: A1·A2·A4·B1 [[references/spx-audit-log-analysis]]. 외부 IdP 그룹 자동 동기화는 본 문서 §5에서 정전화하고 A4·C1(Get Started)이 인용.

---

# 2. 화면 진입 — 워크스페이스 설정 사이드바 2탭

A1 §9.3에서 박제한 3탭(멤버·부서·감사 로그) 중 **멤버·부서 두 탭**이 본 챕터 범위. 감사 로그는 B1에서 다룸.

| 탭 라벨 | 컴포넌트 | 노출 조건 |
|--------|---------|---------|
| **멤버** (`settings.members`) | `members-page/index.tsx` | `isCurrentWorkspaceManager == true` (owner/admin) |
| **부서** (`settings.departments`) | `departments-page/index.tsx` | 동일 — owner/admin 전용 |

> 매뉴얼 톤: "관리자 이상만 사이드바에서 이 두 탭이 보입니다. 일반 멤버에게는 메뉴 자체가 노출되지 않습니다."

---

# 3. 부서 페이지 — `departments-page/`

## 3.1 부서 목록 표

- 컬럼: **부서 코드 / 부서명 / 상태 / 멤버 수 / 작업**
- 정렬: **활성 부서 우선**, 그 안에서 부서명 가나다순(`index.tsx` line 47-53)
- 빈 상태: "부서가 없습니다" placeholder
- 상태 배지: **활성**(녹색), **비활성**(회색)

## 3.2 작업 버튼 — 코드와 HDD 차이 (A1 §C7 확정)

| 작업 | 코드 노출 | HDD 명세 |
|------|---------|---------|
| 멤버 관리 | ✅ 노출 | ✅ 명시 |
| 활성/비활성 토글 | ✅ 노출 | (명시 없음 — soft delete 패턴) |
| **편집**(이름·코드 수정) | ❌ `className="hidden"` (`index.tsx` line 177) | ✅ "수정" 동작 명시 |
| **삭제** | ❌ `className="hidden"` (`index.tsx` line 191) | ✅ "삭제" 동작 명시 |

→ **매뉴얼 작성 정책**:
- 편집·삭제 진입점이 막혀 있으므로 본 챕터에서는 "**비활성으로 정리**"만 안내
- 부서 자체 삭제는 백엔드에 남아 있음 — INV-13(멤버·소유 리소스 모두 없을 때만 삭제) — 운영자 부록에 한 줄 안내 후보
- 이름/코드 변경은 Keycloak 그룹 sync(§5)로 자동 갱신되므로 UI 편집이 불필요한 설계로 추정 — A4·운영자 가이드에서 확정

## 3.3 부서 생성 — `department-form-modal.tsx`

- 진입: 헤더 우측 **부서 추가** 버튼 (`isCurrentWorkspaceManager` 전용)
- 입력: 부서 코드(고유) + 부서명
- 검증:
  - 코드 중복 시 백엔드 409 `department_code_conflict` → 토스트 에러
  - 빈 코드/이름 시 disabled
- 저장 후 활성 상태로 신설, 목록 즉시 갱신

## 3.4 활성/비활성 토글

- `useUpdateDepartment` → `PUT /workspaces/current/rbac/departments/<id>` 로 `is_active` 플립
- **활성→비활성 효과**:
  - 부서 페이지 목록에는 그대로 노출(상태만 비활성)
  - 멤버 페이지 부서 드롭다운/cell에서 **숨겨짐**(`department-cell.tsx` line 61-64 `filter(d => d.is_active)`)
  - 기존 ACL·소유 부서 매핑은 그대로 유지 — 평가 시 "비활성 부서가 owner인 리소스"는 단축 경로 매칭 안 되는 케이스 존재
- 매뉴얼 톤: "부서를 비활성화하면 신규 멤버 배정이 불가능해지지만, 기존 권한·소유 관계는 그대로 유지됩니다."

## 3.5 멤버 관리 모달 — `members-modal.tsx`

부서별 멤버 추가·제거를 한 화면에서 처리.

- 좌측: **현재 멤버** 목록 (해당 부서 소속자, 카운트 표시, 각 행 우측에 "제거" 버튼)
- 우측: **추가 가능 멤버** 목록 (해당 부서에 미소속자 + 검색창)
  - 다른 부서에도 속한 멤버는 옆에 **소속 부서 배지** 표시 (다중 멤버십 정보) — line 250-257
  - 멀티 선택 → 헤더의 **선택 N명 추가** 버튼
- 추가 호출: `useAssignMemberDepartment({ action: 'assign', departmentId })` 순차 fan-out
- 제거 호출: `useAssignMemberDepartment({ action: 'unassign' })` — **주의: 이 액션은 해당 멤버의 모든 부서 멤버십 제거** (`department_service.unassign_member` 전체 drop)

> **운영상 주의**: 멤버 모달의 "제거" 버튼이 호출하는 `unassign` 액션은 **그 멤버가 속한 모든 부서에서 제거**한다(코드 line 113-116). 부서별 부분 제거는 멤버 페이지 부서 셀(§4.3 set 액션)로만 가능. 매뉴얼 안내 시 이 차이를 명확히 — "이 모달의 제거는 모든 부서에서 해제, 한 부서만 빼려면 멤버 페이지에서 부서를 다시 선택"으로 톤.

> **🔴 모달 부제 stale 문구 (제품 i18n 버그, 2026-06-16 발견)**. `rbac.department.membersModal.description` = "이 부서에 속한 구성원을 관리합니다. **한 명은 한 부서에만 속할 수 있어요**" (EN: "Each member can belong to only one department") — **코드와 모순**. 모달 코드는 완전히 가산형 다중 멤버십 기반: line 40-43(현재 멤버는 다른 부서에도 속할 수 있음을 명시), line 83-86(`action: 'assign'` = 이동 아닌 추가), line 225-229 주석 *"adding them here no longer 'moves' them (it's additive)"*, line 250-257(다중 소속 배지). 즉 **단일→다중 전환 후 i18n 문구만 미갱신된 잔재**. **문서는 "여러 부서에 속할 수 있습니다"가 정확** — UI 문구를 따라가지 말 것. 제품 수정 대상: `web/i18n/{ko-KR,en-US}/common.json` 둘째 문장 삭제/교정. 챕터 본문은 영향 없음.

---

# 4. 멤버 페이지 — `members-page/`

## 4.1 멤버 목록 표

- 컬럼: **이름 / 부서 / 마지막 활동 / 역할**
- 페이징: 50명/페이지 (`PAGE_SIZE = 50`)
- 필터: 검색(이름·이메일) + 역할 셀렉트 + 부서 셀렉트 (셋 다 변경 시 1페이지 리셋)
- 부서 필터에는 **"미배정"** 옵션 별도 (DEPT_FILTER_NONE)
- "you" 표시 — 본인 행에 표기

## 4.2 역할 드롭다운 — 기존 Dify 모델 유지

- 옵션: 소유자 / 관리자 / 편집자 / 일반 멤버 / 지식 관리자 (마지막은 `datasetOperatorEnabled` 시)
- 본 챕터 범위는 부서 운영 중심이라 역할 변경 흐름은 Dify 원본 멤버 페이지 챕터에서 다룸(A4와 통합 후보)
- 소유자 행은 별도 처리:
  - 본인이 소유자면 **소유권 이전 모달** 진입 버튼만 (`isAllowTransferWorkspace` 분기)
  - 본인이 관리자(소유자 아님)면 소유자 행은 read-only

## 4.3 부서 셀 — `department-cell.tsx`

멤버 행마다 부서를 직접 편집할 수 있는 다중 선택 셀.

- **트리거 라벨 규칙**:
  - 부서 0개 → "미배정"
  - 부서 1개 → 부서명
  - 부서 N개 → "○○부 외 N-1개"
- 클릭 → 체크박스 드롭다운 (활성 부서만 노출 — INV: 비활성 부서 신규 배정 불가)
- 적용 시 호출: `useAssignMemberDepartment({ action: 'set', departmentIds: [...] })`
  - `set` 액션 = wholesale replace (백엔드 `set_member_departments` diff 기반 갱신)
  - 빈 배열로 적용하면 unassign과 동일
- "전체 해제" 버튼 별도 — 현재 부서가 1개 이상일 때만 노출

## 4.5 헤더 부서 셀렉터 — 앱·지식 목록 부서 필터 (2026-06-05 신규 분석)

> 별도 정전 분석본: [[references/spx-department-filter-analysis]]. 본 절은 요약 + 인용 위치만.

스튜디오 헤더 좌측의 **부서 셀렉터**(`web/app/components/header/account-dropdown/department-selector/`)로 앱·지식 목록을 부서별로 좁힐 수 있다. Dify 원본의 워크스페이스 전환 자리를 spx 단일 워크스페이스 정책에 맞춰 **부서 컨텍스트 전환으로 대체**한 추가 기능.

| 항목 | 요약 (정전: 별도 분석본 §3·§4) |
|------|--------------------------|
| 영향 범위 | **앱·지식 목록만** (도구 미적용) |
| 노출 분기 | OWNER/ADMIN = 전체+활성 부서 / 비-admin 다중 = 본인 부서들 / 비-admin 단일 = read-only / 비-admin 0개 = 미렌더 |
| 백엔드 동작 | `owner_department_id` 일치만 좁힘. 권한 평가는 별도 layer (A1 §7 그대로) |
| 영속 | `localStorage["spx:selected-department"]` (브라우저 단위) |
| 챕터 본문 톤 | [[references/spx-department-filter-analysis#6-챕터-본문-톤-예시]] 그대로 옮김 |

## 4.4 편집 가능 조건

- `editable = isCurrentWorkspaceManager && account.role !== 'owner'`
- 즉 **소유자(Owner) 본인 행의 부서 셀은 편집 불가** — 관리자조차 못 바꿈
- 일반 멤버에게는 부서 셀이 read-only 텍스트로만 노출

---

# 5. Keycloak 그룹 ↔ 부서 자동 동기화 — A1·A2 정정 박제

> **표기 주의**(§0 가이드): 본 절은 분석본의 코드 정전 영역이라 **"Keycloak" 직접 표기를 유지**한다. 챕터 본문 작성 시 사용할 추상화 예시는 §5.6에 별도로 박는다.

## 5.1 정책 — mirror (wholesale replace), HANDOVER §6.1 정정

> **A1 §C4 정정**: HANDOVER §6.1·§6.4는 "가산형(additive) — 기존 멤버십 삭제하지 않음"이라고 박혀 있으나, **실제 코드는 mirror(wholesale replace)** 정책을 사용한다.

근거 코드 (`account_service.py` line 363-365):

```python
DepartmentService.set_member_departments(
    tenant_id, account_id=account.id, department_ids=target_dept_ids
)
```

→ JWT `groups` claim의 현재 상태가 정전(source of truth). 그룹에서 빠지면 **부서 멤버십도 자동 제거**된다.

| 시나리오 | 동작 |
|---------|------|
| 새 그룹에 가입 | 부서 멤버십 자동 추가 (필요 시 부서 자동 생성) |
| 그룹에서 빠짐 | 다음 로그인 시 부서 멤버십 자동 제거 |
| 그룹명 변경 | UUID 매칭 유지 → 부서는 그대로, `code`만 새 leaf name으로 갱신 |
| 부서 표시명(`name`) | **운영자가 부서 페이지에서 관리** — Keycloak이 덮어쓰지 않음 |

## 5.2 안전장치 — `groups` claim 부재 시 no-op

`groups` claim 자체가 JWT에 없으면(예: KC 클라이언트의 group-membership mapper 미설정), **wipe 회피**를 위해 동기화 자체가 no-op (`account_service.py` line 243-245).

명시적 빈 리스트(`"groups": []`)는 "그룹 없음" 의도로 받아들여 전체 해제 진행.

→ 운영 안내: KC에서 그룹 매퍼가 꺼지면 부서 동기화가 멈춘다. 디버깅 시 mapper 적용 상태 먼저 확인.

## 5.3 매칭 우선순위

1. **UUID 매칭** (`keycloak_group_id` 컬럼) — 안정적
2. **code 매칭** (leaf 세그먼트) — legacy/Admin API 일시 실패 fallback. 매칭 성공 시 lazy하게 UUID backfill
3. **부서 자동 생성** — 매칭 안 되면 신규 부서 row 생성 (`keycloak_group_id` 기본 채움)

## 5.4 동기화 시점 — 로그인 콜백

- `KeycloakCallbackApi.get()` → `provision_default_workspace_for_keycloak()` → `_sync_department_from_keycloak_groups()`
- **실시간 sync 없음** — 그룹 변경 후 본인이 한 번 더 로그인해야 반영
- 매뉴얼 톤: "Keycloak에서 그룹을 변경하면, 다음 로그인 시 부서가 자동으로 갱신됩니다."

> **🔴 부서 없음 ≠ 로그인 차단 (2026-06-16 코드 검증)**. `provision_default_workspace_for_keycloak()`(`account_service.py` L132~206) 순서: ① 계정 upsert(sub 기준) → ② 공유 워크스페이스 합류(`normal`) → ③ 부서 동기화. **부서 동기화는 ②까지 commit 후 실행**되고, 코드 주석(L202-203)이 *"Runs after the account + workspace commit **so a sync failure cannot block login**"*로 명시. `groups` claim 부재 시 ③은 no-op(L226-230) → 사용자는 **부서 0개(DB `spx_departments` 멤버십 없음)라도 계정·워크스페이스 멤버십은 생성되어 로그인됨**.
> - 즉 "부서 없으면 로그인 불가"는 **spx-agent 동작이 아님**. 실제로 로그인이 막힌다면 그건 **외부 IdP(Keycloak) 인증 플로우/정책**(그룹 미소속자 토큰 거부) 차원이며 spx-agent 코드·매뉴얼 범위 밖.
> - 매뉴얼 표기: 로그인 차단은 언급하지 않는다. 부서 자동 설정만 중립 표기("부서 소속은 로그인 시 연동된 계정 관리 시스템 기준으로 자동 설정됩니다"). 챕터 본문 적용 완료(`workspace/departments/readme.mdx` §소개).

## 5.5 동기화가 깨지는 케이스 — 운영자 진단 동선

| 증상 | 원인 후보 | 조치 |
|------|---------|------|
| 사용자가 그룹 가입 후 로그인했는데 부서가 없음 | KC mapper 미설정(`groups` claim 부재) → no-op | KC 클라이언트 mapper 활성화 |
| 그룹에서 뺐는데 부서 멤버십이 남음 | 사용자가 아직 재로그인 안 함 | 사용자가 재로그인하면 자동 정리. 즉시 정리는 부서 셀에서 수동 set |
| KC Admin API 일시 장애 후 매칭 누락 | UUID 매칭 실패, code fallback 동작 | 다음 로그인 시 자동 backfill |
| 부서명이 KC와 다름 | `name`은 운영자 관리 필드 — 의도된 동작 | 정책 안내. 표시명 변경은 부서 페이지에서 |

## 5.6 챕터 본문 추상화 표기 예시 (§0 가이드 적용본)

> 챕터 작성 시 본 절의 표기를 그대로 옮긴다. §5.1~§5.5는 분석본 영역(코드 정전)이라 "Keycloak" 직접 표기 유지, 본 절은 챕터 본문(`workspace/departments/readme.mdx`, 7번 작업으로 폴더 이동 완료) 영역의 권장 톤. 5번 작업(2026-06-04)에서 본 절 추상화 톤 그대로 적용됨.

### 5.6.1 도입(§5.1 대응)

> spx-agent에 SSO로 로그인하면, 로그인 시점에 사용자가 속한 **외부 시스템(예, Keycloak)의 그룹**이 자동으로 부서에 매핑됩니다.

### 5.6.2 동작 정책(§5.1 표 대응)

| 시나리오 | 동작 |
|---------|------|
| 새 그룹에 가입 | 부서 멤버십 자동 추가. 매칭되는 부서가 없으면 부서 자동 생성 |
| 그룹에서 탈퇴 | 다음 로그인 시 부서 멤버십 자동 제거 |
| 그룹명 변경 | 부서는 그대로 유지(내부 ID로 매칭), 부서 코드만 새 그룹명으로 갱신 |
| 부서 표시명 | 부서 페이지에서 관리합니다. **외부 시스템이 덮어쓰지 않습니다** |

### 5.6.3 안전장치(§5.2 대응)

> 외부 시스템에서 그룹 정보를 전달하지 않는 경우(예, 그룹 매퍼 미설정), 동기화는 작동하지 않습니다. 기존 부서 멤버십은 그대로 유지됩니다.

### 5.6.4 매칭 우선순위(§5.3 대응)

> 외부 시스템의 그룹은 안정적 내부 식별자(예, UUID)로 부서와 연결됩니다. 그룹명이 바뀌어도 같은 부서로 매칭되며, 매칭 실패 시 부서 코드로 fallback합니다.

### 5.6.5 동기화 시점(§5.4 대응)

> 외부 시스템에서 그룹을 변경한 효과는 사용자가 다음에 로그인할 때 반영됩니다. 즉시 반영이 필요하면 워크스페이스 관리자가 멤버 페이지의 부서 셀에서 수동으로 갱신할 수 있습니다.

### 5.6.6 진단 동선(§5.5 대응)

| 증상 | 확인할 점 |
|:-----|:---------|
| 사용자가 그룹 가입 후 로그인했는데 부서가 없음 | 외부 시스템에서 그룹 정보가 전달되는지(그룹 매퍼 설정) |
| 그룹 변경 후 부서가 그대로 | 사용자가 재로그인했는지 확인. 즉시 정리는 부서 셀에서 수동 set |
| 부서명이 외부 시스템의 그룹명과 다름 | 의도된 동작 — 부서 표시명은 부서 페이지에서 관리 |

---

# 6. 권한 모델과 부서 운영의 접점

A1·A2가 "운영자에게 요청"으로 위임한 동선을 본 문서에 흡수.

## 6.1 부서 이동/변경 시 권한 변화

- 사용자 직접 ACL은 그대로 — 부서 이동과 무관 (HDD §19.2)
- 이전 부서 대상 ACL은 사용자가 그 부서 멤버 아니게 되면 자동 무효 — 별도 작업 불필요
- 새 부서 대상 ACL은 자동 적용 — 별도 작업 불필요
- **단, 자동 적용까지 시간차**: 부서 셀 set 액션은 즉시 반영. KC sync는 다음 로그인 후
- 매뉴얼 톤: "부서를 바꾸면 새 부서로 부여된 권한이 즉시 적용되고, 이전 부서로 부여된 권한은 자동으로 사라집니다. 별도 권한 재설정이 필요하지 않습니다."

## 6.2 부서 멤버 변경 후 지식 검색 권한이 안 맞을 때 — A2 §3.4 트리거

A2에서 보강: 부서 멤버를 추가/제거해도 **그 부서가 소유 부서인 지식의 네이티브 동기화는 자동 트리거되지 않음**. 결과적으로:

- 신규 멤버가 추가됐는데 그 부서 소유 지식이 RAG 검색에서 안 보임
- 제거된 멤버가 여전히 검색에서 그 지식을 봄

조치 — 워크스페이스 RBAC 강제 재동기화 API 호출:
- 모든 지식 drift 확인: `GET /workspaces/current/rbac/datasets/consistency`
- 강제 재동기화: `POST /workspaces/current/rbac/datasets/<id>/sync`

매뉴얼 톤: "부서 멤버를 변경한 후 지식 검색 결과가 일치하지 않으면, 관리자가 지식 정합성 검증을 실행해 정리합니다." (A2 FAQ → A3 동선으로 통합 인용)

---

# 7. 운영자 진단 도구 — 매뉴얼 운영자 부록 후보

## 7.1 사용자별 권한 목록 (운영자 조회)

A1 FAQ "본인의 DENY 행 진단" 동선의 정답.

- 엔드포인트: `GET /workspaces/current/rbac/accounts/<account_id>/permissions`
- 응답: 해당 사용자가 직접 부여받은 모든 ACL 행 (사용자 ACL + 그 사용자가 속한 부서 ACL 분해)
- 현 UI 노출: **없음** — 운영자가 API 또는 추후 운영자 대시보드에서 조회
- 매뉴얼 톤: "특정 사용자가 권한 문제를 보고하면 관리자가 사용자별 권한 목록 API로 부여 내역을 확인합니다."

## 7.2 부서별 소유 리소스

- 엔드포인트: `GET /workspaces/current/rbac/departments/<id>/resources`
- 용도: 부서 비활성/삭제 전 정리해야 할 소유 리소스 식별

## 7.3 워크스페이스 ownership 스냅샷

- 엔드포인트: `GET /workspaces/current/rbac/resources/<type>/ownerships`
- 용도: 리소스별 소유 부서·소유 계정 일괄 조회

## 7.4 감사 로그 — B1 위임

- `GET /workspaces/current/rbac/audit-logs` 페이지네이션 조회
- 도메인 액션명: `department.create`, `department.member.assign`, `department.member.unassign` 등
- 상세는 [[references/spx-audit-log-analysis]] (B1)에서

---

# 8. 컨펌 트랙 영향 + 2026-06-04 결정 반영

## 8.1 2026-06-04 회의 결정 직접 반영 (결정 5·6)

| 결정 | 본 산출물 반영 위치 | 처리 결과 |
|------|------------------|----------|
| **결정 5** — KC 추상화 전역 규칙 #6 신설 | §0 신설 + §5.6 챕터 본문 추상화 예시 | ✅ 본 갱신에서 적용 |
| **결정 6** — `workspace/team-members-management` 변환 → 본 챕터로 대체 | frontmatter `purpose` + §1 본 문서 두 역할 명시 + §10 A4 인용 매핑 | ✅ 본 갱신에서 적용. A4 [[references/spx-workspace-analysis]] §3.3이 변환 경위를 인용 |

## 8.2 결정 6 변환 처리 — 매뉴얼 구성 영향

원본 `workspace/team-members-management.mdx`(5종 역할 Accordion·이메일 초대·다중 워크스페이스)는 spx-agent에서 다음과 같이 변환된다:

| 원본 절 | 변환 처리 |
|--------|---------|
| 5종 역할 Accordion | 본 문서 §4.2 한 줄 위임 + A1 [[references/spx-app-permissions-analysis#7.3-워크스페이스-역할별-capability-매트릭스]] |
| 이메일 초대 절차 | **삭제** — 외부 시스템 mirror로 멤버 추가 (§5) |
| 다중 워크스페이스 안내 | **삭제** — spx-agent 단일 워크스페이스 정책 |
| 멤버 관리(제거·역할 변경) | 본 문서 §4 멤버 페이지 + §3.5 멤버 관리 모달 |
| Access Patterns | A1 [[references/spx-app-permissions-analysis]] 위임 |

→ 결과적으로 본 분석본의 §3·§4·§5는 원본의 변환 결과물이고, Phase 4 본격 작성 시 원본 페이지 자리에 본 챕터를 배치한다.

## 8.3 잔여 컨펌 영향

| 항목 | 현재 가정 | 본 챕터 반영 위치 |
|------|---------|----------------|
| Keycloak 그룹 탈퇴 시 자동 멤버십 해제 정책 | 코드는 mirror, HANDOVER §10.3 "정책 미결"로 표기 | §5.1에서 mirror 정책 확정 박제. 운영 정책으로 굳히려면 [[decisions]] 박제 필요 |
| 부서 편집/삭제 UI 노출 여부 | hidden 적용 (의도 추정) | §3.2. 추후 노출 결정 시 본 챕터 + UI 변경 |
| 멤버 페이지 역할 변경 흐름 (Dify 원본) | A4 통합 | §4.2 한 줄 위임 |

---

# 9. 인터리브 임시 작성 메모 — 부서 관리 챕터 1p 골격

> A3 마감 직후 1p 임시 작성으로 검증. A1·A2 챕터와 형태 정렬.

```
# 사용자/부서 관리

## 접근 권한
- 워크스페이스 관리자 이상만 사이드바 노출 (멤버·부서 두 탭)

## 부서 페이지
- 부서 목록(코드·이름·상태·멤버 수)
- 부서 추가
- 멤버 관리 모달
- 활성/비활성 토글 (편집·삭제는 본 버전 미노출)

## 멤버 페이지
- 멤버 목록(이름·부서·마지막 활동·역할)
- 검색·역할·부서 필터
- 부서 셀: 다중 선택, 적용 시 일괄 갱신
- 소유자 행은 부서 편집 불가

## Keycloak 로그인 자동 동기화
- 로그인 시점에 KC 그룹 → 부서 자동 미러
- 그룹 가입/탈퇴는 다음 로그인 시 자동 반영
- 부서 표시명은 운영자가 관리(KC가 덮어쓰지 않음)

## 자주 묻는 질문
- "부서를 옮겼는데 이전 부서 권한이 자동으로 빠지나?" → 예
- "부서에 새 멤버 추가했는데 그 부서 소유 지식이 검색에 안 잡힌다" → 관리자가 정합성 재동기화
- "Keycloak에서 그룹 뺐는데 부서 멤버십이 남는다" → 다음 로그인 시 자동 정리
- "부서를 삭제하고 싶다" → 비활성으로 정리 (편집·삭제 UI는 현재 미노출)
```

---

# 10. A4·B1·C1이 본 문서를 인용하는 방식

| 후속 산출물 | 본 문서에서 인용할 섹션 |
|------------|----------------------|
| [[references/spx-workspace-analysis]] (A4) | §1 본 문서가 team-members-management 변환 정전임을 명시 + §8.2 변환 처리 표 인용 + §2 사이드바 2탭·§4.2 역할 매트릭스 한 줄 위임 — Workspace 11p 전수 재검토 시 멤버 페이지 절은 본 문서로 위임 |
| [[references/spx-audit-log-analysis]] (B1) | §7.4 감사 로그 도메인 액션명 (`department.*`) — 13 collectors 이벤트 종류 정리 시 |
| [[references/spx-get-started-analysis]] (C1) | §5.4 KC 로그인 콜백 동기화 시점·§5.6 추상화 표기 — Quick Start "로그인 시 부서 자동 배정" 한 줄 |

## 10.1 챕터 본문 작성 시 본 문서 인용 위치

본 문서는 분석본이므로 챕터 본문에 그대로 옮기지 않는다. 다음 매핑 사용:

| 챕터 본문에서 다룰 주제 | 본 문서 참조 위치 |
|--------------------|----------------|
| 부서 페이지 UI 흐름 | §3 — 원문 그대로 한국어 매뉴얼 톤으로 풀어 작성 |
| 멤버 페이지 부서 셀·필터 | §4 |
| **헤더 부서 셀렉터** (앱·지식 필터링) | **§4.5 요약 + [[references/spx-department-filter-analysis#6-챕터-본문-톤-예시]] 그대로 옮김** (2026-06-05 신규 분석) |
| 외부 IdP 그룹 동기화 (사용자 노출) | **§5.6 추상화 표기 예시 그대로 옮김** (Keycloak 직접 노출 금지) |
| 외부 IdP 동기화 동작 정확도(운영자 부록) | §5.1~§5.5 인용 시 "Keycloak"을 "외부 시스템(예, Keycloak)"으로 치환 |

---

# 11. A1·A2 산출물에 역으로 반영해야 할 정정

본 분석에서 발견된 사항 중 A1·A2에 박제된 내용 정정 후보:

- **A1 §C4 정정**: HANDOVER가 "가산형"이라고 적었으나 실제 코드는 mirror — A1 §C4를 "코드는 wholesale replace mirror로 변경. HANDOVER §6.1 정전 갱신 필요"로 보강
- A1 §13.6에 적은 "다중 멤버십을 평가에선 OR로 처리하지만 소유 부서는 1개" — 그대로 유효 (멤버십 set은 다중, owner_department_id는 단일)

→ A1 산출물 §2 차이 표 C4 행 갱신 + §13에 본 정정 언급. (이번 작업에서 처리)

---

# 12. 후속 작업 체크리스트 (A3 마감용)

## 12.1 초안 작성 (2026-06-04)

- [x] `departments-page/`·`members-page/`·`members-modal.tsx`·`department-cell.tsx` UI 코드 검증
- [x] `department_service.py` 5개 메서드(create/update/delete/assign/set/unassign) 검증
- [x] `account_service._sync_department_from_keycloak_groups` 정책 정확화(mirror 확정)
- [x] 워크스페이스 RBAC 부서 엔드포인트 표면 확인
- [x] §3.5 운영상 주의(`unassign` = 전체 해제) 등 사용성 함정 박제
- [x] A1·A2 모호 동선의 정답 위치 매핑
- [x] A4·B1·C1 인용 매핑
- [x] A1 §C4 정정 — KC 동기화 mirror 정책 박제
- [x] **인터리브 챕터 1p**: `ko/use-spx-agent/workspace-management/departments/readme.mdx` 작성·`sidebars.js` "워크스페이스 관리" 카테고리 등록·빌드 통과

## 12.2 이사님 결정 5·6 반영 갱신 (2026-06-04, "남은 작업 1번")

- [x] §0 KC 추상화 표기 가이드 신설 (정책·사유·보존 영역·본 문서 내부 적용)
- [x] §5 도입부에 분석본 영역 표기 정책 박스 추가
- [x] §5.6 챕터 본문 추상화 표기 예시 신설 — 도입·동작 정책·안전장치·매칭 우선순위·동기화 시점·진단 동선 6개 절
- [x] §1 본 문서가 team-members-management 변환 정전임을 명시 (결정 6)
- [x] §8 컨펌 트랙을 §8.1(결정 5·6 직접 반영) / §8.2(변환 처리 표) / §8.3(잔여 컨펌)으로 재구성
- [x] §10 A4 인용 매핑에 §1·§8.2 추가, §10.1 챕터 본문 인용 위치 매핑 신설
- [x] frontmatter `status` 갱신·`status_history`·`related_decisions` 신설

## 12.3 별도 작업으로 분리 (본 갱신 범위 외) — 2026-06-04 완료

- [x] **챕터 본문 KC 추상화 적용** — `workspace/departments/readme.mdx`의 9개 직접 노출 행을 §5.6 예시로 교체. 헤딩 "## Keycloak 그룹 동기화" → "## 외부 시스템 그룹 동기화", FAQ·앵커 일관 갱신 (5번 작업, 2026-06-04)
- [x] **폴더 이동** — `workspace-management/departments/` → `workspace/departments/` (Option α 적용. Option G→α 진화 반영) (7번 작업, 2026-06-04)
- [ ] 후속(deferred to Phase 4 본격 작성): Dify 원본 멤버 페이지 역할 변경 흐름과 통합, 운영자 부록(API 도구 모음) 정리, 부서 편집·삭제 UI 노출 정책 결정