---
title: 권한 설정 — 권한 모델 코어 분석
phase: Phase 4 / 클러스터 A / A1
status: 완료 (2026-06-02), 챕터명 변경 반영 (2026-06-04), 결정 트랙 영향 정리 (2026-06-04 갱신)
audience: spx-agent 사용자 매뉴얼 작성자 (Phase 4 챕터 집필자)
chapter_name_history:
  - 5/29~6/2: "앱 권한 설정"
  - 6/4 이후: "권한 설정" (앱·지식·도구 공통 권한 모델이라 "앱" 한정 표기 제거)
file_name_note: 파일명은 `spx-app-permissions-analysis.md`로 보존 (코드 폴더명 `app-permissions/`와 일관). 챕터 노출명만 "권한 설정"으로 변경
status_history:
  - 2026-06-02 초안: HDD↔코드 차이 9건(C1~C9), 도메인 모델·가시성·ACL·소유권·역할 매트릭스 박제. 인터리브 챕터 1p 작성·빌드 통과
  - 2026-06-02 보강: §5.4(사용성 제약)·§9.2(가시성 게이트 구분 표) 신설, §11(한국어 매핑) 박제
  - 2026-06-04 (A3): C4 KC 동기화 정책 정정 — HANDOVER 기재(가산형)와 다르게 코드는 mirror(wholesale replace) 정책
  - 2026-06-04 (4번): 결정 트랙 영향 정리(§1.1 신설), §13 6항목 상태 점검, A2·A3·A4가 본 문서를 인용한 위치 역방향 매핑(§12) 보강
purpose: |
  spx-agent 부서형 RBAC의 권한 모델 코어를 사용자 매뉴얼 관점으로 정전화.
  본 문서는 클러스터 A의 정전(canonical) 산출물 — A2(Knowledge 권한),
  A3(사용자/부서 관리), A4(Workspace 재검토)는 본 문서의 모델·용어·UI 흐름을
  재사용하고 도메인 특이사항만 추가한다.
source_documents:
  - C:\Users\Administrator\Projects\spx-agent\dify_rbac_hdd_design.md  # 명세 (v3.0.0, 2026-05-07)
  - C:\Users\Administrator\Projects\spx-agent\RBAC_HANDOVER.md         # 인수인계 (2026-05-18)
related_decisions:
  - 결정 1 (2026-06-04 회의) — Marketplace/플러그인 전체 제거. 본 문서 직접 영향 없음 (§1.1)
  - 결정 3 (2026-06-04 회의) — MCP 유지·번역. §3.1 도구 정의의 "MCP provider 통합" 표현 유지
  - 결정 5 (2026-06-04 회의) — KC 추상화 전역 규칙 #6. 본 문서는 코어 정전이라 분석본 내부는 영향 없음. 챕터 본문은 A3 [[references/spx-departments-management#0-표기-가이드-—-kc-추상화-전역-규칙-6]] 따름
  - 결정 6 (2026-06-04 회의) — team-members-management 변환. 본 문서 직접 영향 없음 — A3·A4 인용 매핑만 명확화
code_verified_at: 2026-06-02
---

# 1. 본 문서의 위치

본 문서는 **사용자 매뉴얼 집필을 위한 분석 노트**다. 코드 명세는 [dify_rbac_hdd_design.md](file:///C:/Users/Administrator/Projects/spx-agent/dify_rbac_hdd_design.md) (HDD)와 [RBAC_HANDOVER.md](file:///C:/Users/Administrator/Projects/spx-agent/RBAC_HANDOVER.md) (HANDOVER)에 박제되어 있다. 본 문서는 다음을 재정리한다:

1. **HDD와 실제 코드의 차이**(현시점 진실은 코드)
2. **사용자가 알아야 할 수준의 동작 모델**(UI 흐름·라벨·시나리오)
3. **A2~A4 분석이 인용할 정전 요소**(권한 모델 코어, 가시성 4단계, ACL 표면, 소유권)

> **클러스터 A 진행 순서**: A1(본 문서) → A2 [[references/spx-knowledge-permissions]] ∥ A3 [[references/spx-departments-management]] → A4 [[references/spx-workspace-analysis]]. 위 산출물들은 본 문서의 **§3 도메인 모델 / §4 가시성 / §5 ACL / §7 권한 판정 / §9 권한 탭 UI**를 인용 기준으로 사용한다.

> **인접 분석**: B1 감사 로그는 [[references/spx-audit-log-analysis]], 워크스페이스 KPI는 [[references/spx-dashboard-analysis]].

## 1.1 2026-06-04 이사님 결정 영향

본 문서는 권한 모델 코어 정전(canonical)이라 결정 트랙 영향이 직접적으로 들어오지 않는다. 영향 정리:

| 결정 | 본 문서 영향 | 처리 |
|------|-------------|-----|
| **결정 1** — Marketplace/플러그인 전체 제거 | 없음 — 권한 모델은 Marketplace와 무관 | grep 0건 확인 |
| **결정 2** — 외부 연결(outbound) 제거 | 없음 — 권한 모델은 외부 연결과 무관 | (A2·A4가 직접 영향) |
| **결정 3** — MCP 유지 | §3.1 "MCP provider 3종 통합" 표현 유지 — 도구 정의는 그대로 | 표현 검토 완료 |
| **결정 4** — inbound 3건 유지 | 없음 — 권한 모델은 inbound/outbound 분류와 무관 | (A2 §7이 직접 영향) |
| **결정 5** — KC 추상화 (전역 규칙 #6) | 본 분석본 내부는 영향 없음 (코드 정전이라 식별자 보존). **챕터 본문은** A3 [[references/spx-departments-management#0-표기-가이드-—-kc-추상화-전역-규칙-6]] 적용 대상 | 챕터 본문 갱신은 별도 5번 작업 |
| **결정 6** — team-members-management 변환 → A3 챕터 | 없음 — 권한 모델 코어는 변환 대상이 아님. A3·A4 인용 매핑(§12)에서 변환 위치 명시 | §12 보강 |
| **결정 7~10** — CI/CD·문서 대상·사이드바 Option α(2026-06-04 G→α 진화)·rename | rename은 chapter_name_history에 박제(2026-06-04). 사이드바는 7·8번 작업(2026-06-04)으로 Option α 적용 완료 | frontmatter 박제 |

→ 본 문서가 직접 갱신 대상인 결정은 없음. **본 문서를 인용하는 챕터 본문 작업(5번·6번·7번)에서 결정 5·6 적용**.

---

# 2. HDD vs 코드 차이 — 작성 시 코드 기준 적용

HDD는 v3.0.0(2026-05-07) 시점 명세이고 이후 코드가 갱신됐다. **사용자 매뉴얼은 코드 기준으로 작성**한다. 차이는 다음과 같다.

| # | 영역 | HDD 기술 | 실제 코드 | 매뉴얼 작성 시 |
|---|------|---------|---------|---------------|
| C1 | 멤버십 카디널리티 | INV-1 "사용자 1인 1부서" | **다중 멤버십 허용** (마이그레이션 `a7b8c9d0e1f2`). 평가는 OR semantics — `actor_dept_ids` 집합, 부서 ACL은 IN 절로 일괄 조회 | "사용자는 **여러 부서**에 속할 수 있다"로 기술. UI 부서 선택은 단일 선택 화면이 일부 남아 있음(visibility-section의 owner 부서) |
| C2 | Admin override 범위 | INV-4 "Admin은 manage_permission만 override" | **Admin도 OWNER와 동일하게 전 액션 override** (`authorization_service.can()` line 216 `if role in (OWNER, ADMIN): return True`) | Admin = Owner와 동급으로 모든 권한 통과. 매뉴얼은 "관리자(Admin) 이상은 모든 리소스를 자유롭게 다룰 수 있다" |
| C3 | DUPLICATE 액션 | HDD §3.2 매트릭스에 없음 | App에 한해 `Action.DUPLICATE` 추가, 매트릭스 셀 `(APP, DUPLICATE)` = OWNER/ADMIN/EDITOR. `CREATOR_DEFAULT_ACTIONS`에도 포함 | App 권한 탭 액션 체크박스에 "복제(duplicate)" 항목 노출. 매뉴얼에 8개 액션 표기 |
| C4 | Keycloak 그룹 동기화 | HDD에 미기재 | 구현 완료 — KC 로그인 시 JWT `groups` claim 파싱 → 부서 자동 sync. **mirror(wholesale replace) 정책** — 그룹에서 빠지면 다음 로그인 시 부서 멤버십도 자동 제거. HANDOVER §6.1의 "가산형(additive)" 표기는 코드와 불일치 → A3 [[references/spx-departments-management]] §5에서 mirror 확정 | 본 문서는 코어만 다룸. 자세한 운영 흐름은 A3 §5에서 |
| C5 | 워크스페이스 RBAC 사이드바 위치 | HDD §12.2 "Permissions 탭 1개" | **`멤버` / `부서` / `감사 로그` 3개 탭으로 분리** (`account-setting/index.tsx`, owner/admin 전용) | "사용자/부서 관리"는 단일 탭이 아니라 두 탭(멤버 / 부서). 챕터 구성 시 반영 — A3 참조 |
| C6 | `Action.PUBLISH` Dataset/Tool | HDD "정의 없음" | 코드 매트릭스에 셀 없음(미정의 = 모든 role 거부, INV-15 보존) | "발행" 액션은 App 전용으로 안내 |
| C7 | 부서 편집/삭제 버튼 | HDD §12.2 "admin/owner 전용 CRUD" | 코드는 편집·삭제 버튼에 `className="hidden"` 적용 — 화면에 미노출. 대신 **활성/비활성 토글**과 **멤버 관리**만 노출 | "부서는 비활성화로 운영을 정리한다(편집·삭제 UI는 막혀 있음)"로 기술. 삭제는 백엔드에 남아 있으나 화면 진입점 없음 |
| C8 | Dataset Document 세분화 ACL | HDD §19.4 미구현(Phase 9 후보) | 여전히 미구현 | A2에 "지식 문서 단위 ACL은 현재 미지원" 한 줄 |
| C9 | `expires_at` 만료 | HDD §19.5 컬럼만 존재 | 여전히 평가 미반영 | UI에 만료 옵션 노출 안 함, 매뉴얼에서도 다루지 않음 |

> **현행 박제 책임**: 위 차이 중 C1·C2·C3·C4는 HANDOVER §10.1에 이미 기록됨. C5·C7은 새로 식별. 추후 HDD/HANDOVER 갱신은 본 문서 범위 밖.

---

# 3. 도메인 모델 — 사용자 매뉴얼용 어휘

## 3.1 보호 대상(리소스) 3종

| 매뉴얼 표기 | 내부 값 | 포괄 범위 |
|------------|--------|----------|
| **앱**(App) | `app` | Chat / Completion / Workflow / Agent / Chatbot 모두 — spx-agent의 모든 앱 형태 |
| **지식**(Knowledge) | `dataset` | 지식 베이스(RAG dataset). 문서 단위 세분화는 미지원 |
| **도구**(Tool) | `tool` | Custom API 도구 + Workflow-as-Tool + MCP provider 3종 통합 |

> 매뉴얼 톤은 영문 키워드 병기를 피하고, 가시성/액션 라벨은 한국어로 옮긴다. 한·영 매핑은 [[conventions]] 용어집 후속 갱신.

## 3.2 액션 — 사용자 화면 라벨 기준 8종

| 라벨(체크박스) | 내부 값 | 의미 |
|---------------|---------|------|
| 조회 | `view` | 목록·상세 조회 |
| 편집 | `edit` | 설정·내용·플로우 수정 |
| 삭제 | `delete` | 리소스 삭제 |
| 실행 | `execute` | App: 메시지 전송 / 지식: 검색 테스트 / 도구: 호출 |
| 발행 | `publish` | App 전용 — 사이트/API 활성, 워크플로 배포 |
| 복제 | `duplicate` | **App 전용** — 원본을 읽어 사본 생성 |
| 권한 관리 | `manage_permission` | 가시성·ACL 변경 |
| 소유권 이전 | `transfer` | 소유 부서·소유자 변경 |

> `create`는 권한 부여 모달에 노출되지 않는다(per-resource 액션이 아니라 워크스페이스 역할 매트릭스가 결정 — INV-14).

## 3.3 권한 주체(Principal) 2종

| 매뉴얼 표기 | 내부 값 | 의미 |
|------------|--------|------|
| 사용자 | `user` | 특정 계정에 직접 부여 |
| 부서 | `department` | 부서의 모든 활성 멤버에게 일괄 부여 |

권한 부여 모달은 두 탭으로 분리(`user` / `department`) — 동시 선택 불가, 한 번에 한 종류만.

## 3.4 효과(Effect) 2종

- **허용**(`allow`, 기본) — 명시적 허용
- **거부**(`deny`) — 명시적 차단. 생성자 묵시 권한까지 무력화. 현재 UI는 `allow`만 제출(grant 모달 코드 line 141 `effect: 'allow'`) — `deny`는 API 호출로만 가능하며 운영자가 의도적으로 박을 때만 사용

> 매뉴얼 작성 시: "거부(DENY) 권한은 일반 UI에 노출되지 않으며, 의도적 차단이 필요할 때만 관리자가 API로 등록한다"는 한 줄 가이드만 둔다.

---

# 4. 가시성(Visibility) 4단계 — 핵심 개념

리소스 한 건의 공개 범위. **App / 지식 / 도구 모두 동일한 4단계**. UI는 라디오 버튼 1개 그룹(`visibility-section.tsx`).

| 매뉴얼 표기 | 내부 값 | 묵시 부여(자동 권한) | 소유 부서 입력 |
|------------|--------|------------------|-----------------|
| **비공개** | `private` | 없음 — 생성자만 + 명시 ACL 대상자만 | 선택 불필요 |
| **부서 공개** | `department` | 소유 부서 멤버 전원에게 **조회·실행** 자동 부여 | **필수** — 소유 부서 1개 지정 |
| **사용자 지정** | `custom` | 없음 — 명시 ACL 대상자만 (부서 단축 경로 없음) | 선택 가능(권장) — 소유 부서 표시용 |
| **워크스페이스 공개** | `workspace` | 워크스페이스 전원에게 **조회·실행** 자동 부여 | 선택 불필요 |

> **자동 부여되는 액션은 `view`·`execute` 두 가지뿐** — `edit`·`delete`·`publish`·`manage_permission`·`transfer`는 어떤 가시성에서도 자동 부여되지 않는다. 이 액션이 필요하면 §5 명시 ACL로 부여 (HDD F-13 footgun, INV-7).

## 4.1 시나리오로 본 가시성 동작

> 매뉴얼 본문에 그대로 인용 가능한 시나리오. HDD §7.6 결정표(E1–E19)에서 사용자 시각으로 추린 것.

| 시나리오 | 결과 | 근거 |
|---------|------|------|
| **비공개 앱**을 비창작자가 열려 함 | ❌ 차단 | private + ACL 없음 |
| 비공개 앱을 **창작자**가 편집 | ✅ 통과 | 창작자 묵시 권한(`CREATOR_DEFAULT_ACTIONS`) |
| 비공개 앱에 본인 대상 `edit/deny` 행이 있는 상태에서 창작자가 편집 | ❌ 차단 | 명시 거부는 묵시 권한을 무력화 (INV-6) |
| **부서 공개**(소유=영업) 앱을 **영업** 멤버가 조회 | ✅ 통과 | 부서 단축 경로(`view`) |
| 같은 앱을 영업 멤버가 **편집** | ❌ 차단 | 부서 단축 경로는 `view`·`execute`만 |
| 같은 앱을 **인사** 멤버가 조회 | ❌ 차단 | 부서 불일치 |
| 부서 공개 앱(소유 부서=미배정) | ❌ 차단(같은 액션 어떤 멤버도 자동 권한 없음) | 양쪽 부서 비-NULL 필요 (INV-7) |
| **워크스페이스 공개** 앱을 일반 멤버가 조회 | ✅ 통과 | workspace 단축 경로 |
| 같은 앱을 일반 멤버가 편집 | ❌ 차단 | workspace 단축 경로는 `view`·`execute`만 |
| **사용자 지정** 앱에 본인 `edit/allow` 행이 있음 | ✅ 통과 | 사용자 명시 ACL |
| 사용자 지정 앱에 본인 `edit/allow` + 본인 부서 `edit/deny`가 동시 | ✅ 통과 | 사용자 ACL이 부서 ACL을 가린다 (INV-17, HDD F-14) |

## 4.2 가시성 변경 UI 동작 (`visibility-section.tsx`)

- 라디오 4종 + 소유 부서 드롭다운(부서/사용자 지정에서만 표시)
- 부서 공개로 전환 시 소유 부서 필수 — 미선택 상태로 저장 누르면 inline 검증 에러
- 저장 시 호출: `PUT /apps/<id>/visibility` (App), `/datasets/<id>/visibility`, `/tool-provider/<type>/<id>/visibility`
- `manage_permission` 권한이 없으면 라디오 disabled, 저장 버튼 disabled — 페이지 자체는 read-only로 렌더(403 대신, `index.tsx` line 33 주석 참조)
- 지식의 경우 visibility 변경은 자동으로 Dify 네이티브 `only_me/all_team_members/partial_members`로 동기화됨 (§8 참조) — 사용자는 따로 신경 쓸 필요 없음

---

# 5. 명시 ACL — 사용자/부서별 권한 부여

가시성 단축 경로로 부족할 때(예: 부서 멤버 중 일부만 편집 가능, 타 부서 부장에게도 조회 권한). UI는 권한 탭 하단 **ACL 섹션**(`acl-section.tsx`)에서 관리.

## 5.1 권한 부여 흐름

1. **권한 부여 버튼** → 모달(`grant-permission-modal.tsx`)
2. 모달 상단 탭: **사용자 / 부서** 중 하나 선택
3. 검색 + 멀티 선택 (이미 권한이 있는 주체는 disabled, "이미 부여됨" 배지)
4. 액션 체크박스 8종(App)·7종(지식·도구, duplicate 없음) — 기본 `view` 체크
5. 저장 → 선택한 N명 × 액션 조합으로 백엔드 `POST /<rt>/<id>/permissions` 호출

> **백엔드 API는 1요청 = 1주체** 구조라 N명 선택 시 N회 순차 호출. 한 명이라도 실패하면 거기서 중단하고 토스트로 "○○에서 실패" 표시(`acl-section.tsx` line 133-148).

## 5.2 표시: 주체별 1행으로 그룹핑

ResourcePermission 테이블에는 `(리소스, 주체, 액션)` 단위로 행이 저장되지만 UI는 **`(주체)`별 1행으로 묶어** 액션을 칩으로 나열한다. 컬럼:

| 주체 | 유형 | 권한 | 작업 |
|------|------|------|------|
| 인사부 | 부서 | 조회, 실행 | 편집 / 취소 |
| 김OO | 사용자 | 편집 | 편집 / 취소 |
| 박OO | 사용자 | 권한 관리 | 편집 / 취소 |

- **편집** 버튼 → 같은 모달이 "수정 모드"로 열림. 주체는 잠금, 액션 체크박스를 토글해 add/remove diff 계산 → grant + per-entry revoke 직렬 호출
- **취소** 버튼 → 확인 다이얼로그 → `(주체)` 전체 bulk revoke 한 번에 호출(`/permissions/principal/<type>/<id>` DELETE)
- 부서 행이 사용자 행보다 위 — 권한 한눈 스캔 용이 (`acl-section.tsx` 정렬 line 99-105)
- 행에 `deny` 효과가 하나라도 있으면 칩이 **빨간 배지**로 표시 (운영자가 의도적 차단을 즉시 식별)

## 5.3 모달 보조 동작

- `permission-context.can_manage_permission == false`이면 **권한 부여 버튼 미노출, 행 작업 버튼 미노출** (read-only 모드)
- 비활성 사용자(워크스페이스에서 제거됐지만 ACL 행은 남음): 라이브 principals 목록에 없어도 행에 표시 — 행 편집 시 모달이 최소 record 합성(`acl-section.tsx` line 120-126)
- 다이얼로그 폭 600px — 액션 그리드 2열 + 푸터 여유

## 5.4 사용성 제약 — 매뉴얼에 명시 필요 (인터리브 보강)

- **사용자·부서 동시 선택 불가**: 모달 상단 탭(`user` / `department`)이 배타적이라 한 번의 부여 호출에 두 종류가 섞이지 않는다. 두 종류 모두 부여하려면 모달을 두 번 연다.
- **`actions` 체크박스 최소 1개**: API 페이로드 `actions: list[str] = Field(..., min_length=1)` — 빈 체크 상태로 저장 불가. UI에선 저장 버튼 disabled.
- **기본 체크는 조회만**: 사용자가 "조회만 주려고 한" 케이스를 가장 흔한 출발점으로 본 디폴트. 권한 부여 모달은 항상 `view` 1개 체크된 상태로 열린다.

---

# 6. 소유권(Ownership) — 누가 소유자인가

리소스마다 하나씩 존재하는 `ResourceOwnership` 행. **lazy 생성** — 권한 인터랙션이 일어나기 전까지 만들지 않는다.

## 6.1 생성 시점

| 리소스 | 자동 stamp 시점 |
|--------|----------------|
| 앱 | 첫 가시성 변경 또는 첫 ACL 부여 시 `ResourceOwnershipService.ensure()` |
| 지식 | 동일 |
| 도구 | **생성 즉시 자동 stamp** — DB after_insert 리스너(`tool_ownership_listener.py`). tool 생성과 동일 트랜잭션. 실패 시 swallow(INV-10) — 권한 페이지 진입 시 lazy 보정 |

> **lazy의 의미**: ownership 행이 없는 리소스는 권한 판정이 **워크스페이스 역할 매트릭스만 통과하면 자동 통과**(INV-5). 부서 RBAC 도입 전에 만들어진 리소스가 그대로 동작하는 근거. 매뉴얼에서는 굳이 표면화하지 않고 "권한 설정을 한 번이라도 만지면 본격 적용된다" 정도로만 안내.

## 6.2 소유자 카드 — 권한 탭 최상단(`OwnerCard`)

- 표시: 생성자 계정명 + 이메일
- 표시 원천: `visibility.owner_account_id` (visibility API 응답에 포함)
- 클릭 동작 없음 — 정보 카드

## 6.3 소유 부서 변경 = visibility-section에서 직접

- 소유 부서 드롭다운에서 다른 부서 선택 → 저장하면 `owner_department_id` 업데이트
- **소유 계정 변경 UI는 미노출** — 백엔드 payload에 `owner_account_id`가 있지만(`UpdateVisibilityPayload`) 프론트 visibility-section은 보내지 않는다. transfer 액션은 별도 흐름이 필요한데 현재 UI는 미구현
- 매뉴얼에서는 "소유 계정 이전은 현 시점 UI 미지원, 운영자가 백엔드로 처리"로 안내

---

# 7. 권한 판정 흐름 — 사용자 시각에서

사용자가 "왜 이 리소스에 못 들어가지?"를 물을 때 답할 수 있는 단순화 모델. 정확한 알고리즘은 HDD §7.

## 7.1 4단계 판정(요약)

```
[1] 같은 워크스페이스인가? + 워크스페이스 역할이 있는가?
       → 둘 다 통과 시 다음
[2] 워크스페이스 역할이 Owner / Admin인가?
       → 그렇다면 모두 통과 (Owner와 Admin 동급, 코드 기준)
[3] 워크스페이스 역할이 이 (리소스 종류, 액션)에 대해 허용되는가?
       (역할 매트릭스, 예: Normal은 앱 편집 안 됨)
       → 통과 못 하면 차단
[4] 리소스에 권한 설정이 한 번이라도 만져졌는가?
       → 아니면 통과 (lazy 폴백)
       → 만져졌다면 ACL 평가:
            (a) 내가 생성자 + 액션이 묵시 권한 묶음에 있고 거부 행이 없음 → 통과
            (b) 가시성 단축 경로 (workspace / department) → 통과
            (c) 내 사용자 명시 ACL 존재 → 그 effect로 결정 (DENY 우선)
            (d) 내 부서 명시 ACL 존재 → ALLOW 하나라도 있으면 통과
            (e) 어느 것도 매칭 안 되면 차단
```

## 7.2 매뉴얼에 그대로 옮길 수 있는 한 문장

> **"내가 만든 리소스는 별도 권한 설정 없이도 다룰 수 있고, 다른 사람이 만든 리소스는 그 리소스의 공개 범위와 권한 설정에 따라 접근이 결정된다. 워크스페이스 관리자(Admin) 이상은 모든 리소스에 접근할 수 있다."**

## 7.3 워크스페이스 역할별 capability 매트릭스

> 본 절은 매뉴얼에 "역할별 기본 권한" 한 페이지로 옮길 수 있는 형태. **코드 기준**(`_BASE_ROLE_CAPABILITY`).

### 앱(App)

| 액션 | OWNER | ADMIN | EDITOR | NORMAL | DATASET_OPERATOR |
|------|:-:|:-:|:-:|:-:|:-:|
| 조회·실행 | ✓ | ✓ | ✓ | ✓ | — |
| 생성·편집·삭제·발행·복제·권한관리 | ✓ | ✓ | ✓ | — | — |
| 소유권 이전 | ✓ | ✓ | — | — | — |

### 지식(Dataset)

| 액션 | OWNER | ADMIN | EDITOR | NORMAL | DATASET_OPERATOR |
|------|:-:|:-:|:-:|:-:|:-:|
| 조회·실행 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 생성·편집·삭제·권한관리 | ✓ | ✓ | ✓ | — | ✓ |
| 소유권 이전 | ✓ | ✓ | — | — | — |

### 도구(Tool)

| 액션 | OWNER | ADMIN | EDITOR | NORMAL | DATASET_OPERATOR |
|------|:-:|:-:|:-:|:-:|:-:|
| 조회·실행 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 생성·편집·삭제·권한관리 | ✓ | ✓ | ✓ | — | — |
| 소유권 이전 | ✓ | ✓ | — | — | — |

> Dataset Operator는 도구·앱과 무관, 지식만 라이프사이클 관리 가능. Normal은 어떤 리소스도 라이프사이클을 만지지 못한다.

---

# 8. 지식(Dataset)의 특이사항 — A2로 위임

지식 권한은 본 코어 모델에 두 가지 layer가 더 얹힌다:

1. **Dify 네이티브 권한 모델과 자동 동기화** — `only_me / all_team_members / partial_members` + `DatasetPermission` 행. 커스텀 ACL 변경 시 자동 sync.
2. **`partial_members`는 항상 creator 포함** (INV-8) — 생성자가 visibility 잘못 설정해도 본인이 잠기지 않음.
3. **view + ALLOW 행만 투영** (INV-9) — Dify 네이티브는 액션 구분 없음.
4. **drift 검증 API** 존재 — `/workspaces/current/rbac/datasets/consistency` (운영자용).

상세는 [[references/spx-knowledge-permissions]](A2 산출물) 참조. 본 문서는 코어 모델만 박제하고 위임한다.

---

# 9. 권한 탭 UI 위치 정리 — 챕터별 사용 위치

## 9.1 리소스별 권한 탭 진입점

| 리소스 | 사이드바 라벨 | 라우트 패턴 | 컴포넌트 |
|--------|-------------|-----------|---------|
| 앱 | "권한" 탭 (앱 상세 좌측) | `/app/[appId]/permissions/page.tsx` | `web/app/components/app-permissions/` |
| 지식 | "권한" 탭 (지식 상세 좌측) | `/datasets/[datasetId]/permissions/page.tsx` | `web/app/components/dataset-permissions/` |
| 도구 | 도구 상세 시트 내 "권한" 섹션 | (provider 시트 내 패널) | `web/app/components/tool-permissions/` |

> 세 컴포넌트의 페이지 헤더는 모두 동일 i18n 키(`appPermissions.pageTitle`) — 코드는 같은 라벨을 공유한다. 매뉴얼에서도 일관된 "권한 설정" 헤더로 가는 것이 자연스럽다.

## 9.2 탭/페이지 가시성 게이트

> 두 가지 다른 가시성 메커니즘이 존재 — 매뉴얼에서 혼동 피해야 함.

| 화면 | 노출 조건 | 메커니즘 |
|------|---------|---------|
| **앱/지식/도구 상세의 "권한" 메뉴** | 페이지 진입 자유 | `permission-context` 응답의 `can_manage_permission == true`이면 편집 가능, false면 같은 페이지가 **read-only 모드**로 렌더 (`app-permissions/index.tsx` line 33). 403 반환 안 함 — URL 직접 입력으로도 정보 확인 가능 |
| **워크스페이스 설정 사이드바의 "권한 관리" 탭** (구 RBAC 페이지) | `useManageableApps()` 결과가 1건 이상일 때만 노출 | 본인이 권한 관리할 수 있는 앱이 하나도 없으면 탭 자체 숨김. HDD §12.2 |
| **워크스페이스 설정 사이드바의 "멤버"/"부서"/"감사 로그" 탭** | `isCurrentWorkspaceManager == true` (owner/admin) | `account-setting/index.tsx` line 71, 123 |

→ 매뉴얼 톤 권장: "권한 메뉴는 본인이 해당 앱에 대해 권한 관리 권한을 가질 때 편집 가능합니다. 없으면 읽기 전용으로 표시됩니다."

App 라우트는 sidebar에서 `Permissions` 메뉴를 통해 진입, 지식은 상세 페이지 좌측 메뉴.

## 9.3 워크스페이스 RBAC 진입점 — 사이드바 3탭

> HDD가 가정한 단일 "Permissions" 탭은 코드에서 **3개 탭으로 분리**됐다 (C5). 본 정보는 A3 챕터에 인용.

| 탭 라벨(i18n) | 컴포넌트 | 역할 |
|-------------|---------|------|
| `settings.members` | `members-page/` | 워크스페이스 멤버 목록 + 부서 배정 셀 |
| `settings.departments` | `departments-page/` | 부서 목록 + 활성/비활성 토글 + 멤버 관리 모달 |
| `settings.auditLog` | `audit-log-page/` | RBAC 감사 로그(13 collectors, B1 인용) |

3탭 모두 **`isCurrentWorkspaceManager == true` (owner/admin) 일 때만 노출**(`account-setting/index.tsx` line 71, 123).

부서 페이지는 다음 액션만 노출:
- 부서 생성 (+ 새 부서 모달)
- 부서 활성/비활성 토글 — 운영 중 부서를 비활성으로 정리
- 부서 멤버 관리 모달 — 활성 부서별 멤버 추가/해제
- **편집·삭제 버튼은 코드상 `className="hidden"` 적용**으로 숨김 — 운영 중 실수 방지

---

# 10. 운영자용 RBAC API — 매뉴얼 "참고" 정도

본 절은 일반 사용자 문서에는 노출 안 함. 운영자(spx-agent 운영팀) 매뉴얼이나 부록에 옮길 후보.

| 기능 | 엔드포인트 |
|------|-----------|
| 워크스페이스 부서 CRUD | `/workspaces/current/rbac/departments[/<id>]` |
| 멤버 부서 배정 (set/assign/unassign) | `/workspaces/current/rbac/members/<account_id>/department` |
| 본인이 관리 가능한 앱 | `/workspaces/current/rbac/manageable-apps` |
| 부서별 소유 리소스 | `/workspaces/current/rbac/departments/<id>/resources` |
| 사용자별 권한 목록 | `/workspaces/current/rbac/accounts/<id>/permissions` |
| 워크스페이스 ownership 스냅샷 | `/workspaces/current/rbac/resources/<type>/ownerships` |
| 지식 정합성 검증 (drift) | `/workspaces/current/rbac/datasets/consistency` |
| 지식 강제 재동기화 | `POST /workspaces/current/rbac/datasets/<id>/sync` |
| 감사 로그 페이지네이션 | `/workspaces/current/rbac/audit-logs` |

---

# 11. 인터리브 임시 작성 — 권한 설정 챕터 1p (2026-06-02 완료, 6/4 명칭 갱신)

본 분석을 토대로 권한 설정 챕터 1p 임시 작성 → 누락 검증 완료. 결과물 위치 및 등록 상태:

- 작성 파일: `ko/use-spx-agent/workspace/permissions/readme.mdx` (6번 작업으로 이동 완료, 2026-06-04. 이전 경로 `ko/use-spx-agent/app-permissions/`)
- 사이드바 등록: `sidebars.js` "권한 설정" 카테고리 (6/4 rename, 구 "앱 권한 설정")
- 빌드 검증: `npm run build` 성공 (broken link 경고는 Phase 4에서 추가될 페이지 관련 기존 항목)

## 11.1 챕터 구성 (작성된 골격)

1. 소개 — 가시성 + ACL 두 축 + 생성자/관리자 자동 접근 (§3·§7.2 인용)
2. 권한 탭 진입 — 좌측 사이드바 + read-only 모드 (§9.2 인용)
3. 가시성 범위 — 4종 옵션 표 + 자동 부여 권한 명시 (§4 인용)
4. 자원 권한 — 시나리오·부여 절차 Steps·액션 8종 표·목록 편집 (§5 인용)
5. 자주 묻는 질문 4건 — 시나리오 기반 답변 (§4.1·§6.3·§7.2 인용)

## 11.2 챕터 작성 중 확정된 한국어 매핑

> 분석 §14 후속 작업 일부 자동 완료. conventions glossary 일괄 갱신은 Phase 4 본격 작성 시 진행.

| 영역 | 영문 | 한국어 |
|------|------|-------|
| Visibility scope | private | 비공개 |
| Visibility scope | department | 부서 공개 |
| Visibility scope | custom | 사용자 지정 |
| Visibility scope | workspace | 워크스페이스 공개 |
| Action | view | 조회 |
| Action | edit | 편집 |
| Action | delete | 삭제 |
| Action | execute | 실행 |
| Action | publish | 발행 |
| Action | duplicate | 복제 |
| Action | manage_permission | 권한 관리 |
| Action | transfer | 소유권 이전 |
| Principal type | user | 사용자 |
| Principal type | department | 부서 |
| Effect | allow | 허용 |
| Effect | deny | 거부 |
| Owner Card | — | 소유자 카드 |
| Permission tab | — | 권한 탭(앱 상세) / 권한 메뉴 |
| 모달 액션 버튼 | Grant permission | 권한 부여 |
| 행 액션 | Edit | 편집 |
| 행 액션 | Revoke | 취소 |

## 11.3 임시 작성 중 보강된 분석 항목

- §5.4 신설 — 사용자·부서 동시 선택 불가, `actions` 최소 1개, 기본 체크는 조회만
- §9.2 갱신 — 권한 메뉴 vs 워크스페이스 설정 권한 탭의 가시성 메커니즘 구분 표
- §11(본 절) — 챕터 1p 작성 완료 및 한국어 매핑 확정

## 11.4 챕터 1p 작성 중 추가 식별된 후속 확인 항목

다음은 본 문서가 아닌 별도 검증·결정이 필요한 항목 — Phase 4 본격 작성 또는 컨펌 트랙에서 처리.

- **앱 상세 좌측 메뉴의 i18n 키** — 매뉴얼에 "권한" 메뉴 라벨이라 적었으나, 코드의 사이드바 라벨이 정확히 무엇인지(i18n 키 + 한국어 값)는 추후 확인. App 권한 탭 i18n은 `appPermissions.pageTitle`이지만 메뉴 진입 라벨은 별도.
- **"거부(DENY) 권한이 본인을 막는" 시나리오의 운영 안내 수위** — 챕터 FAQ에 "권한 관리자에게 확인 요청"으로 적었지만, 일반 사용자가 어떻게 본인의 DENY 행 존재를 진단하는지 매뉴얼 동선은 미정. 운영자 가이드와 통합 검토 필요.
- **소유권 이전(transfer) 액션의 UI 동선 부재** — 챕터에서 "운영자에게 요청"으로 처리. transfer 액션이 매트릭스에 있고 ACL로 부여도 가능한데, 실제 사용자가 transfer를 행사하는 UI가 visibility-section의 owner 부서 드롭다운 하나뿐임. 본격 작성 시 transfer 액션의 매뉴얼적 역할 재검토 필요(매뉴얼에 노출할지, 운영자 부록으로 빼는지).

---

# 12. A2~A4가 본 문서를 인용하는 방식

각 후속 분석은 본 문서를 다음과 같이 참조한다 (산출물 작성 시 [[references/spx-app-permissions-analysis#섹션]] 헤딩 링크 사용 가능).

## 12.1 순방향 — 본 문서가 정전인 항목

| 후속 산출물 | 본 문서에서 인용할 섹션 | 추가로 다룰 항목 |
|------------|----------------------|----------------|
| [[references/spx-knowledge-permissions]] (A2) | §3 도메인 모델·§4 가시성·§5 ACL·§7.3 지식 매트릭스·§9.1 탭 진입 | Dify 네이티브 모델 동기화·INV-8/9, partial_members 구성 규칙, 결정 2·4 외부 연결 처리 |
| [[references/spx-departments-management]] (A3) | §3.3 주체·§4 가시성(부서 공개 단축 경로)·§7.3 역할 매트릭스·§9.3 사이드바 3탭 | 부서 CRUD UI 흐름·멤버 배정 모달·결정 5 KC 추상화 가이드·결정 6 team-members 변환 |
| [[references/spx-workspace-analysis]] (A4) | §2 HDD↔코드 차이(특히 C2 Admin override·C4 KC sync)·§7.3 역할 매트릭스·§9.3 사이드바 3탭 | Personal Account 가정 정정·API Extension 결정 2 삭제·team-members 변환 인용 |

## 12.2 역방향 — 본 문서가 A2·A3·A4의 갱신 결과를 인용하는 지점 (2026-06-04 4번 작업 보강)

> 클러스터 A 진행 과정에서 A2·A3·A4가 새로 박제한 정전 항목 중, 본 문서의 §2 차이·§13 후속이 인용 위치로 가리키는 지점.

| 본 문서 인용 위치 | 후속 산출물의 정전 위치 | 인용 사유 |
|-----------------|----------------------|----------|
| §2 C4 KC 동기화 mirror 정책 (정정 박제) | A3 [[references/spx-departments-management#5-keycloak-그룹-부서-자동-동기화-—-a1-a2-정정-박제]] | HANDOVER 기재(가산형)와 다른 실제 코드 정책 |
| §2 C5 워크스페이스 RBAC 사이드바 3탭 | A3 [[references/spx-departments-management#2-화면-진입-—-워크스페이스-설정-사이드바-2탭]] + B1 (감사 로그 탭) | 멤버·부서·감사 로그 3탭 분리 사실 정전 |
| §2 C7 부서 편집·삭제 hidden | A3 [[references/spx-departments-management#3.2-작업-버튼-—-코드와-hdd-차이-a1-§c7-확정]] | 매뉴얼 정책: 비활성화로만 정리 |
| §13 표 4번 행 (DUPLICATE 미정의) → 매뉴얼 처리 | A2 [[references/spx-knowledge-permissions#5.0-액션-종수-정리-ui-vs-유효]] | UI 8종 vs 유효 6종 (publish·duplicate 미정의) 박제. 본 문서 §3.2 액션 8종 표 + §7.3 매트릭스 인용 |
| §13 표 3번 행 (transfer UI 동선 부재) | A4 [[references/spx-workspace-analysis#3.4-manage-apps-app-management]] | 본격 작성 시 결정 후보 |
| 결정 5 KC 추상화 표기 정책 (챕터 본문 적용 시) | A3 [[references/spx-departments-management#0-표기-가이드-—-kc-추상화-전역-규칙-6]] + A3 §5.6 추상화 표기 예시 | 본 문서는 분석본이라 코드 식별자 보존. 챕터 본문은 추상화 가이드 따름 |
| 결정 6 team-members 변환 처리 | A3 [[references/spx-departments-management#1-본-문서의-위치]] + A3 §8.2 + A4 [[references/spx-workspace-analysis#3.3-manage-members-team-members-management-—-결정-6-변환-확정]] | 원본 페이지 자리에 A3 챕터 배치. 본 문서 인용 매핑의 "Manage Members 전체" 행이 변환 대상 |

---

# 13. 분석 중 발견된 코드 단서 — 매뉴얼·후속에 반영 후보

> 본 문서 작성 중 식별. 일부는 본문에 흡수됐고, 일부는 별도 트랙(컨펌·UI 개선)으로 옮길 후보.
>
> **2026-06-04 4번 작업 상태 점검**: 결정 트랙(Marketplace·MCP·외부 연결)과 본 §13 6항목 간 교차 영향 grep 결과 직접 영향 행은 없음. 본 §13은 권한 모델 코어 자체에 대한 코드 단서이고, 결정 트랙은 본 문서 인용 챕터의 본문 처리에 적용된다.

| # | 코드 단서 | 상태 (2026-06-04 점검) |
|---|----------|----------------------|
| 1 | **부서 편집·삭제 버튼이 숨겨져 있음** (C7) — 의도 확인 필요. 운영 정책상 비활성화로만 정리하는 것인지, 향후 노출 예정인지 | A3 [[references/spx-departments-management#3.2-작업-버튼-—-코드와-hdd-차이-a1-§c7-확정]]에서 매뉴얼 정책 박제 완료. Phase 4 본격 작성 시 노출 정책 결정 |
| 2 | **권한 부여 모달이 `effect: 'allow'` 고정** (`grant-permission-modal.tsx` 호출부) — DENY는 일반 UI에 없음 | 매뉴얼에서 다루지 않음 확정. 운영자 부록에 한 줄 (Phase 4) |
| 3 | **소유 계정 변경 UI 미구현** — payload는 받지만 visibility-section은 보내지 않음. transfer 액션이 매트릭스에는 있으나 UI 동선 없음 | A4 [[references/spx-workspace-analysis#3.4-manage-apps-app-management]]·§11.4 transfer UI 동선 후속 검토에서 다룸. Phase 4 본격 작성 시 결정 |
| 4 | **DATASET·TOOL의 `Action.DUPLICATE` 미정의** (코드 매트릭스에 셀 없음) — App만 복제 액션 | A2 [[references/spx-knowledge-permissions#5.0-액션-종수-정리-ui-vs-유효]]에서 "UI 8종 / 유효 6종(publish·duplicate 미정의)" 박제. 도구 챕터도 동일 패턴 (Tool 7종으로 따로 박제 — Phase 4 본격 작성 시 검증) |
| 5 | **visibility-section의 owner 부서는 단일 선택** — 다중 멤버십을 평가에선 OR로 처리하지만 소유 부서는 1개. 부서 ACL은 멀티 선택 가능 | 매뉴얼 톤: "리소스의 소유 부서는 하나" / "권한 부여 대상 부서는 여러 개 가능"으로 구분 기술 — Phase 4 본격 작성 시 |
| 6 | **`OwnerCard`는 visibility API의 `owner_account_id`를 표시** — `permissions` 응답에는 없는 정보. 권한 탭 진입 시 visibility GET이 항상 호출되는 이유 | 매뉴얼에서 굳이 표면화하지 않음. 운영자/개발자 부록 후보 |

## 13.1 Marketplace/MCP 관련 행 점검 (4번 작업)

- "Marketplace" grep — **0건**. 권한 모델 코어와 무관, 결정 1 영향 없음
- "MCP" grep — **1건** (§3.1 도구 정의의 "MCP provider 3종 통합"). 결정 3 MCP 유지에 따라 표현 그대로 보존
- 결과: §13 6항목 중 결정 트랙으로 해소·삭제되는 행 없음. 4번 작업 "Marketplace/MCP 관련 행 정리"는 noop 확인.

---

# 14. 후속 작업 체크리스트 (A1 마감용)

## 14.1 초안·인터리브 (2026-06-02)

- [x] HDD·HANDOVER 정독, 코드 검증 (`authorization_service.py`, `constants.py`, `app-permissions/`, `dataset-permissions/`, `tool-permissions/`, `rbac_enforcement.py`, `tool_providers_permission.py`, `workspace/rbac.py`, `departments-page/`, `account-setting/index.tsx`)
- [x] 코드↔HDD 차이 9건 정리 (§2)
- [x] 도메인 모델·가시성 4단계·ACL·소유권·권한 판정·역할 매트릭스를 사용자 매뉴얼용 어휘로 재정리
- [x] 워크스페이스 RBAC 사이드바 실제 구성(3탭) 박제
- [x] A2~A4 인용 매핑 표 작성
- [x] **인터리브 단계**: 권한 설정 챕터 1p 임시 작성 (`ko/use-spx-agent/app-permissions/readme.mdx`) → 누락 보강 (§5.4·§9.2 갱신, §11 매핑 박제) → 빌드 검증 통과

## 14.2 결정 트랙 영향 정리 (2026-06-04, "남은 작업 4번")

- [x] frontmatter `status` 갱신·`status_history`·`related_decisions` 신설 — A3 mirror 정정·4번 작업 결과 박제
- [x] §1.1 신설 — 결정 1·2·3·4·5·6·7~10이 본 문서에 미치는 영향 표 박제 (대부분 직접 영향 없음, 결정 5는 챕터 본문에 적용)
- [x] §13 6항목 상태 점검 — 표 형식으로 재구성, 각 항목이 A2·A3·A4·Phase 4에서 어떻게 다뤄지는지 박제
- [x] §13.1 Marketplace/MCP grep 결과 박제 — Marketplace 0건, MCP 1건(§3.1 표현 유지), 4번 작업이 noop임을 명시
- [x] §12.2 신설 — 역방향 인용 매핑(A2·A3·A4의 갱신 결과가 본 문서의 어느 절을 보강했는지)
- [x] §3.1 MCP 표현 검토 완료 — 결정 3 유지에 따라 그대로 보존

### 14.2.1 자체 검토 결과 (2026-06-04 본 작업 마감 직전)

- [x] §12.2 인용 위치 표기 — "§13.4 DUPLICATE"·"§13.3 transfer" → "§13 표 4번 행·3번 행"으로 정정 (§13.1 신설 절과 헷갈림 회피)
- [x] frontmatter status_history 4건·related_decisions 4건 일관성 확인
- [x] §2 C4 KC 직접 노출은 §0 KC 추상화 가이드의 "분석본 코드 정전 보존 영역"이므로 의도된 유지 확인
- [x] §13 표·§13.1 절 구분 — 표 6행 + §13.1 grep 결과 별도 절로 명확
- [x] §11.1 작성 파일 경로 stale 점검 — 점검 시점은 Option G 이동 전 + "이동 예정" 명시로 일관 처리. 이후 6번 작업(2026-06-04)에서 Option α 적용으로 실제 이동 완료, §11.1 본문도 동기화

## 14.3 Phase 4 본격 작성 시 처리

- [x] [[conventions]] 용어 글로서리에 본 문서 신규 매핑(§11.2) 반영 — 2026-06-04 일괄 완료 (progress L218 — 권한 모델·로그인 옵션·감사 로그 3 sub-section)
- [x] 챕터 본문 헤더·소개 "권한 설정" 명칭 반영 — 6번 작업(2026-06-04)에서 폴더 이동(`app-permissions/` → `workspace/permissions/`)과 함께 완료
- [ ] §11.4 후속 확인 3건(사이드바 i18n / DENY 진단 동선 / transfer UI 동선)
- [ ] §13 6항목 중 Phase 4 결정 후보 — C7 부서 편집·삭제 노출 정책, transfer UI 동선, owner 부서 단일 vs 부서 ACL 멀티 톤
