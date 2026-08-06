---
title: 부서 필터링 UI — 헤더 드롭다운 분석
phase: Phase 4 / 클러스터 A / 신규 분석 (2026-06-05)
status: 완료 (2026-06-05)
audience: spx-agent 사용자 매뉴얼 작성자 (Phase 4 챕터 집필자)
purpose: |
  스튜디오 헤더 상단의 **부서 선택 드롭다운**(앱·지식 목록을 부서별로 필터링)
  분석. Dify 원본에 없는 spx-agent 추가 기능. A3 사용자/부서 관리 챕터에
  sub-section으로 흡수 + Get Started Quick Start 한 줄 anchor 추가 예정.
parent_document: spx-departments-management.md  # A3 — 부서 운영 정전
related_documents:
  - spx-app-permissions-analysis.md  # A1 — owner_department_id 권한 모델 근거
  - spx-workspace-analysis.md        # A4 — 헤더 영역과의 관계
code_verified_at: 2026-06-05
---

# 1. 본 문서의 위치

본 문서는 [[references/spx-departments-management]](A3)의 **§N 신규 sub-section "부서 필터로 자원 보기"** 정전이다. 헤더 드롭다운 자체가 부서 운영의 일부라 별도 챕터를 만들지 않고 A3에 흡수.

- A1과의 관계: 백엔드 필터링은 A1 [[references/spx-app-permissions-analysis#6-소유권ownership-—-누가-소유자인가]]의 `owner_department_id` 컬럼 기반. 권한 평가가 아니라 **소유 부서 일치**로만 좁힘 (§4)
- A4와의 관계: 헤더 영역은 워크스페이스 단위 UI라 [[references/spx-workspace-analysis]] §3.1 Workspace Overview의 사이드바 메뉴 안내에 한 줄 인용 가능

> **인접 분석**: A1·A3·A4. 대시보드(B2) 부서별 집계와는 별도(대시보드는 자체 데이터 모델).

## 1.1 표기 가이드 (전역 규칙 #6 적용)

본 분석본은 코드 정전이라 식별자 보존(`useSelectedDepartmentStore`, `ActorDepartmentsApi`, `owner_department_id` 등). 챕터 본문 작성 시 외부 IdP 관련 표기는 A3 [[references/spx-departments-management#0-표기-가이드-—-kc-추상화-전역-규칙-6]] 따름.

---

# 2. 화면 위치 — 헤더 좌측 셀렉터

| 항목 | 내용 |
|------|------|
| 컴포넌트 | `web/app/components/header/account-dropdown/department-selector/index.tsx` |
| 노출 위치 | 스튜디오 헤더 좌측(워크스페이스 selector 자리) — Dify 원본의 "워크스페이스 전환" UI를 spx에서 **부서 선택으로 대체** |
| 데이터 훅 | `useActorDepartments` → `GET /workspaces/current/rbac/departments/for-actor` (A3 §10.4 운영자 API와 별개 — 일반 사용자도 호출) |
| 상태 저장 | `useSelectedDepartmentStore` (zustand) + `localStorage` (`spx:selected-department`) — 페이지 새로고침·재로그인 후에도 선택 유지 |
| 셀렉터 라벨(드롭다운 헤더) | i18n 키 `common.departmentSelector.header`, **default "부서"** (※ i18n 등록 누락 — fallback 사용) |
| "전체" 옵션 라벨 | i18n 키 `common.departmentSelector.all`, **default "전체"** (※ 동일 누락) |

> **컴포넌트 주석 인용**: "Header selector that replaces the legacy workspace switcher." — Dify의 워크스페이스 전환 자리를 spx 단일 워크스페이스 정책에 맞춰 부서 컨텍스트 전환으로 대체한 의도가 명확.

---

# 3. 노출 규칙 — 4가지 분기

`useActorDepartments` 응답(`is_admin`·`departments[]`)과 본인 멤버십에 따라 다르게 렌더.

| 역할/멤버십 | 셀렉터 렌더 | 메뉴 옵션 | 비고 |
|-----------|-----------|----------|------|
| **소유자(Owner) / 관리자(Admin)** | 드롭다운 | "전체" + 워크스페이스 활성 부서 전체 | `is_admin=true`. 워크스페이스 전 부서로 자유 컨텍스트 전환 |
| **비-admin · 다중 부서** (2개 이상) | 드롭다운 | 본인 소속 활성 부서들만 (**"전체" 없음**) | 다중 멤버십(A1 §C1 OR semantics 보유) 사용자가 자기 부서 사이에서 전환 |
| **비-admin · 단일 부서** | **read-only 라벨** | (드롭다운 없음) | 부서명만 배지처럼 표시. 전환할 다른 부서가 없으므로 메뉴 없음 |
| **비-admin · 부서 0개** | **렌더 안 함** | (없음) | layout-level redirect `/onboarding/no-department`이 먼저 동작 — 셀렉터 진입 안 됨 |

## 3.1 매뉴얼 톤 권장

> 매뉴얼 본문: "헤더 좌측에 본인이 속한 부서가 표시됩니다. 여러 부서에 속한 경우 드롭다운으로 컨텍스트를 전환할 수 있고, 관리자는 워크스페이스 전체 부서를 선택할 수 있습니다."

---

# 4. 필터링 동작 — 백엔드 인터랙션

## 4.1 영향 범위 — 앱·지식 2종에만 적용

`useSelectedDepartmentStore` 사용처 grep 결과:

| 화면 | 컴포넌트 | 백엔드 파라미터 |
|------|---------|----------------|
| **앱 목록** | `web/app/components/apps/list.tsx` | `owner_department_id` (use-apps.ts L36-38) |
| **지식 목록** | `web/app/components/datasets/list/datasets.tsx` | `owner_department_id` (use-dataset.ts) |
| **도구 목록** | (사용처 없음) | — |

→ **도구는 부서 필터링이 적용되지 않는다**. A1 §3.1에 도구도 `owner_department_id`를 가지는 동일 모델이나, 헤더 셀렉터가 도구 목록 hook에 연결돼 있지 않음.

> 매뉴얼 권장 처리: "부서 선택은 앱과 지식 목록에만 적용됩니다. 도구 목록은 부서 컨텍스트와 무관하게 항상 워크스페이스 전체를 보여줍니다."

## 4.2 백엔드 동작 — `owner_department_id` 일치만 좁힘

`use-apps.ts` 주석 인용:
> "the backend narrows the listing to apps whose `resource_ownership.owner_department_id` matches"

즉 부서 선택 = `WHERE resource_ownership.owner_department_id = :selectedDepartmentId` 추가. **권한 평가가 아니라 소유 부서 매칭**.

| 상황 | 결과 |
|------|------|
| `departmentId === null` ("전체") | 파라미터 미전송 → 백엔드 필터 미적용 → 평소처럼 권한 평가만 (A1 §7) |
| `departmentId === 영업` | `owner_department_id = 영업`인 자원만 목록에 노출 |
| 소유 부서가 NULL인 자원 | 어느 부서를 선택해도 매칭 안 됨 → "전체"에서만 보임 |

> **권한 모델과의 관계**: 필터는 권한 평가 이후(또는 별도)로 추가 좁히기. "본인이 접근 권한 있는 자원" 중에서 "소유 부서가 X인 것"으로 추가 좁힘. 즉 부서 필터가 권한을 우회하지 않으며, 권한 없는 자원이 부서 필터로 갑자기 보이지도 않는다.

## 4.3 "(미지정)" 라벨 — 헤더 셀렉터에는 없음

progress 메모에서 "(미지정)" 의미 확인 항목이 있었으나 **헤더 드롭다운에는 해당 옵션이 없다**. 비슷한 라벨은 다른 화면에 등장:

| 라벨 | 등장 화면 | 의미 |
|------|---------|------|
| **"할당 없음"** (`rbac.department.unassigned`, ko-KR) | 멤버 페이지 부서 셀(A3 §4.3), 부서 셀 트리거 라벨 | 멤버가 어느 부서에도 속하지 않음 |
| **"미배정"** | 멤버 페이지 부서 필터 옵션(A3 §4.1 — `DEPT_FILTER_NONE = '__none__'`), 대시보드 부서별 차트 | 같은 의미 — 부서 미배정 멤버 또는 부서 미설정 자원 |

→ 매뉴얼에서 혼동 방지: 헤더 셀렉터의 "전체"는 "필터 없음"이고, 멤버/대시보드의 "미배정·할당 없음"은 "부서 0개 사용자/자원"이다. 라벨 통일은 권장하나 표기 일관성 보장 안 됨 — 후속 conventions 갱신 후보.

---

# 5. 선택 상태 영속성 — localStorage

| 항목 | 내용 |
|------|------|
| 저장소 | `localStorage["spx:selected-department"]` |
| 형태 | zustand `persist` 미들웨어 — `{departmentId: string | null}` 직렬화 |
| 영속 범위 | 같은 브라우저에서 페이지 새로고침·재로그인 후에도 유지 |
| 초기값 | `null` ("전체") — 첫 진입 시 |
| 워크스페이스 전환 시 | spx 단일 워크스페이스 정책이라 N/A |

→ 매뉴얼 톤: "선택한 부서는 브라우저에 저장되어 다음 접속 시에도 그대로 유지됩니다. 다른 브라우저나 컴퓨터에서 로그인하면 다시 '전체'로 초기화됩니다."

---

# 6. 챕터 본문 톤 예시 (A3 §N에 옮길 추상화 톤)

> 챕터 작성 시 그대로 옮길 수 있는 권장 톤. KC 추상화 가이드 준수.

### 헤더 부서 선택

> 스튜디오 화면 좌측 상단의 부서 셀렉터로 **앱과 지식 목록을 부서별로 좁혀볼 수 있습니다.** 본인이 속한 부서가 표시되며, 클릭하면 다른 부서로 컨텍스트를 전환할 수 있습니다.

### 노출 옵션

| 역할 | 보이는 옵션 |
|:-----|:-----|
| **소유자 / 관리자** | "전체" + 워크스페이스의 모든 활성 부서 |
| **여러 부서에 속한 일반 멤버** | 본인이 속한 부서들 (전체 옵션 없음) |
| **한 부서에 속한 일반 멤버** | 본인 부서명만 표시(전환 메뉴 없음) |

### 필터 동작

> 선택한 부서가 **소유 부서**로 지정된 앱·지식만 목록에 표시됩니다. 본인이 접근 권한을 가진 자원 안에서 추가로 좁히는 방식이라, 권한이 없는 자원이 부서 필터로 갑자기 보이지는 않습니다.

> 도구 목록은 부서 컨텍스트와 무관하게 항상 워크스페이스 전체가 보입니다.

> 선택은 브라우저에 저장되어 다음 접속 시에도 유지됩니다.

### 자주 묻는 질문

- **"전체"에는 보이는데 부서를 선택하면 안 보이는 자원이 있습니다** — 그 자원의 소유 부서가 미설정 상태입니다. 자원의 권한 탭에서 가시성을 바꾸면서 소유 부서를 지정할 수 있습니다 (자세히는 [권한 설정](../permissions))
- **드롭다운에 다른 부서가 안 보입니다** — 본인이 한 부서에만 속해 있거나 관리자 권한이 없는 경우입니다. 다른 부서를 보려면 관리자에게 권한 위임 또는 부서 추가 배정을 요청하시기 바랍니다

---

# 7. A3·Quick Start 적용 위치

## 7.1 A3 sub-section 신설 위치

[[references/spx-departments-management]]의 **§4 멤버 페이지** 다음에 신규 §4.5(또는 §5 외부 시스템 그룹 동기화 앞 §4.5) "헤더 부서 선택" 신설. 본 분석본 §6 톤 그대로 옮김.

A3 §10.1 챕터 본문 인용 매핑 표에 행 추가:
| 챕터 본문에서 다룰 주제 | 본 문서 참조 위치 |
|--------------------|----------------|
| 헤더 부서 선택 셀렉터 | 본 문서 §6 추상화 톤 예시 |

## 7.2 Quick Start (C1) anchor 한 줄

C1 분석 시 추가:
> "스튜디오 화면 좌측 상단의 부서 셀렉터로 본인이 속한 부서를 확인하고, 앱·지식 목록을 부서별로 좁혀볼 수 있습니다. 자세히는 [사용자/부서 관리](../workspace/departments) 참조."

## 7.3 A1 권한 설정 챕터에는 인용만

본 기능은 권한 평가가 아니므로 A1 본문에서 본격 다루지 않음. 단 "왜 부서별로 자원이 다르게 보이나" FAQ가 들어가면 본 문서 인용.

---

# 8. i18n·conventions 후속

- [ ] `common.departmentSelector.header`·`common.departmentSelector.all` i18n 키가 실제 파일에 등록되지 않은 상태(`defaultValue` fallback 사용). en-US/ko-KR i18n에 정식 등록 후속 (Phase 4 본격 작성 시 또는 별도 trivial 작업)
- [ ] **conventions 글로서리에 "헤더 부서 셀렉터" 행 추가** — sub-section 신설 시 함께 (Phase 4 본격 작성 시)
- [ ] "할당 없음" vs "미배정" 표기 통일 검토 — 현재 코드에 두 표기 혼재 (i18n은 "할당 없음", 대시보드 본문은 "미배정"). 사용자 매뉴얼 톤은 **"미배정"으로 통일** 권장 (이미 글로서리에 등록된 친숙한 표기) — conventions 후속

---

# 9. 후속 작업 체크리스트

## 9.1 본 분석 (2026-06-05 완료)

- [x] 헤더 셀렉터 컴포넌트 식별 (`department-selector/index.tsx`) + `useActorDepartments` 훅 추적
- [x] zustand `useSelectedDepartmentStore` + `localStorage` 영속성 박제
- [x] 영향 범위 식별 — 앱·지식 2종만 (도구는 미적용)
- [x] 노출 규칙 4분기(Owner·Admin / 비-admin 다중 / 비-admin 단일 / 비-admin 0개) 박제
- [x] 백엔드 동작 — `owner_department_id` 일치만 좁힘, 권한 평가는 별도 layer
- [x] "(미지정)" 라벨 의미 확인 — 헤더에는 없음, 멤버/대시보드의 "할당 없음·미배정"이 별도 의미
- [x] i18n 라벨 검증 — `defaultValue` fallback 사용 중(미등록 발견)
- [x] 챕터 본문 톤·인용 위치 매핑 (§6, §7)

## 9.2 분석본 갱신 (본 작업 후 진행)

- [ ] A3 [[references/spx-departments-management]] §4.5(가칭) sub-section 박제 — 본 분석본 §6 옮김
- [ ] A3 §10.1 챕터 본문 인용 매핑 표에 "헤더 부서 셀렉터" 행 추가
- [ ] [[3. 프로젝트/spx-agent-docs/docs/scope-mapping]] 사용자/부서 관리 챕터 비고에 "헤더 부서 셀렉터" 명시

## 9.3 Phase 4 본격 작성 시

- [ ] A3 챕터 본문(`workspace/departments/readme.mdx`)에 "헤더 부서 선택" 절 추가 — 본 §6 톤 그대로
- [ ] C1 Quick Start에 한 줄 anchor 추가
- [ ] i18n 정식 등록 — `common.departmentSelector.header/all` 키
- [ ] "할당 없음" vs "미배정" 표기 통일 — conventions 글로서리에서 결정
