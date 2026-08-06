---
title: 지식 권한 — 도메인 특이사항 분석
phase: Phase 4 / 클러스터 A / A2
status: 완료 (2026-06-04), 이사님 결정 2·4 반영 (2026-06-04 갱신)
audience: spx-agent 사용자 매뉴얼 작성자 (Phase 4 챕터 집필자)
purpose: |
  A1에서 정립한 권한 모델 코어 위에 지식(Dataset) 도메인 특이사항만 더한
  분석. 가시성·ACL·소유권·역할 매트릭스의 일반론은 [[references/spx-app-permissions-analysis]]에서
  재사용하고, 본 문서는 그 차이점만 박제한다 — 매뉴얼에서 두 챕터가
  중복 설명 없이 서로 인용할 수 있도록.
status_history:
  - 2026-06-04 초안: 권한 모델 코어 차이점 6건(K1~K6) 박제, 외부 연결 패턴 (a)/(b)/(c) 컨펌 대기
  - 2026-06-04 갱신: 결정 2 (패턴 (a) 외부 import / (b) 외부 KB read-only 삭제 확정) + 결정 4 (패턴 (c) 외부 노출 API 유지·번역 확정)
base_document: spx-app-permissions-analysis.md  # A1 — 정전(canonical) 모델
source_documents:
  - dify_rbac_hdd_design.md §8.2, §10  # 명세
  - RBAC_HANDOVER.md §5.4               # 운영
related_decisions:
  - 결정 2 (2026-06-04 회의) — 외부 연결 제거 (outbound). 지식 외부 import 3건·외부 KB read-only 2건 포함
  - 결정 4 (2026-06-04 회의) — inbound 3건 유지·번역. `maintain-dataset-via-api` 포함
code_verified_at: 2026-06-04
---

# 1. 본 문서의 위치 — A1 차이점 박제

본 문서는 [[references/spx-app-permissions-analysis]](A1)의 **권한 모델 코어를 전제**한다. 가시성 4단계의 일반 동작·ACL 모달 UI·생성자 묵시 권한·역할 매트릭스 일반론은 A1을 그대로 인용하고, 본 문서는 **지식 도메인에서 달라지는 부분만** 다룬다.

차이점 여섯 가지:

| # | 지식만의 특이사항 | 어디서 보강 |
|---|------------------|-------------|
| K1 | **Dify 네이티브 권한 모델과의 자동 동기화** — `only_me / all_team_members / partial_members` + `DatasetPermission` 행. spx 커스텀 ACL이 변경될 때마다 자동으로 다시 계산되어 네이티브 모델에 투영 | §3 |
| K2 | **partial_members는 항상 creator 포함** (INV-8) — 생성자가 가시성을 잘못 잡아도 본인 잠금 방지 | §3.2 |
| K3 | **view + ALLOW 행만 투영** (INV-9) — 네이티브 모델은 액션 구분이 없어 `view` 의도만 신뢰 가능 | §3.3 |
| K4 | **drift 검증·재동기화 API** — 외부 도구가 네이티브 모델을 직접 건드렸을 때 어긋남(drift) 감지·복구 | §4 |
| K5 | **유효 액션 6종**(publish·duplicate 미정의) — UI 권한 부여 모달은 App과 공용이라 8종 체크박스가 모두 보이지만 publish·duplicate는 지식에 적용되지 않음 (INV-15 매트릭스 미정의 = 거부) | §5 |
| K6 | **기존 원본 페이지의 권한 언급 위치** — 부분 수정 격상 대상 3p의 진입점 | §6 |

> **인접 분석**: A1 [[references/spx-app-permissions-analysis]], A3 [[references/spx-departments-management]], A4 [[references/spx-workspace-analysis]], 대시보드 [[references/spx-dashboard-analysis]].

---

# 2. 권한 모델 — A1 그대로 적용되는 부분 (요약 인용용)

> 매뉴얼 집필 시 본 절은 본문에 풀어쓰지 말고, A1 챕터 또는 본 챕터 앞부분에서 참조 한 줄로 처리.

- **가시성 4단계** (비공개·부서 공개·사용자 지정·워크스페이스 공개) → A1 §4
- **소유권 모델** (lazy 생성, 생성자 묵시 권한, 소유 부서) → A1 §6
- **ACL 부여/취소 UI** (사용자/부서 탭, 액션 체크박스, 행 그룹핑, 편집·취소) → A1 §5
- **권한 판정 4단계** + 시나리오 11건 → A1 §7
- **역할 매트릭스 — 지식 열만 인용** → A1 §7.3 "지식(Dataset)" 표
- **HDD↔코드 차이 9건**(특히 C1 다중 멤버십·C2 Admin override) → A1 §2

---

# 3. Dify 네이티브 권한 모델과의 자동 동기화 (K1·K2·K3)

지식의 가장 큰 특이점. 매뉴얼 본문에서 "왜 별도로 신경 쓸 필요 없는가"를 안내하는 근거.

## 3.1 두 모델이 공존하는 이유

- **Dify 네이티브**: 오래된 코드 경로(특히 **RAG 검색**)가 `Dataset.permission` 컬럼(`only_me`/`all_team_members`/`partial_members`)과 `DatasetPermission` 행을 본다. 액션 구분 없음, "조회/검색 가능 여부"만.
- **spx 커스텀 RBAC**: 사용자/부서 ACL, 액션별 권한, 가시성 4단계 — A1의 모델 그대로.
- 둘이 어긋나면 권한 탭에서는 "○○가 조회 가능"인데 RAG 검색에서는 그 ○○가 못 보는 버그가 발생. 그래서 **항상 일치**하도록 spx가 커스텀 ACL → 네이티브 모델로 자동 투영한다.

## 3.2 가시성 → 네이티브 매핑

`DatasetAclSyncService._project()` 결과 (코드 line 186-223).

| spx 가시성 | Dify 네이티브 `Dataset.permission` | partial_members 대상 |
|-----------|----------------------------------|---------------------|
| 비공개(private) | `ONLY_ME` | (없음) |
| 워크스페이스 공개(workspace) | `ALL_TEAM` | (없음) |
| 부서 공개(department) | `PARTIAL_TEAM` | 소유 부서 활성 멤버 ∪ **생성자** |
| 사용자 지정(custom) | `PARTIAL_TEAM` | view+ALLOW 행의 사용자 + (부서 grant인 경우 부서 멤버 펼침) ∪ **생성자** |

> **K2 INV-8 강제** — `_project()` line 218-221: `if ownership.created_by: members.add(ownership.created_by)`. 생성자 ID가 있으면 무조건 추가. 생성자가 visibility 토글 중 본인을 못 보는 사고 방지.

## 3.3 ACL 행 중 어느 것이 투영되는가 — K3 INV-9

코드(line 205-216):

```python
grants = session.scalars(select(ResourcePermission).where(
    ResourcePermission.action == Action.VIEW,
    ResourcePermission.effect == Effect.ALLOW,
    ...
))
```

매뉴얼 톤 권장 설명:

> "지식의 '권한 부여' 모달에서 부여한 권한 중 **조회(view) 허용**으로 등록된 행만 네이티브 검색 권한에 반영됩니다. 편집만 부여하고 조회를 부여하지 않은 사용자는 검색 결과에 노출되지 않습니다(편집을 부여했다면 조회도 함께 부여하는 것이 일반적입니다)."

## 3.4 동기화 시점 — 사용자가 알 필요는 없지만 운영자가 알아야 할 부분

- 가시성 변경 후 (`PUT /datasets/<id>/visibility`)
- ACL grant/revoke 후 (`POST/DELETE /datasets/<id>/permissions[/...]`)
- 부서 멤버 변경 후 — **자동 트리거 없음** (HANDOVER §8.3 미구현 영역). 부서 멤버를 추가했는데 그 부서가 어떤 지식의 소유 부서인 경우, 새 멤버에게 네이티브 검색 권한이 자동으로 안 가는 케이스. 운영자 수동 sync 필요.

## 3.5 동기화 알고리즘 (운영자 부록용)

- `sync_dataset_permission(tenant_id, dataset_id)` — 트랜잭션 안에서 재계산 + wholesale replace (delete all → insert all)
- 작은 set이라 diff하지 않고 통째로 갈아끼움 (HDD §10.1)
- 트랜잭션 분리 회피 — 한 트랜잭션 안에서 끝나므로 중간 상태 노출 없음

---

# 4. drift 검증·재동기화 — K4 (운영자용)

매뉴얼 일반 사용자 문서엔 노출 안 함. 운영자 부록 또는 [[references/spx-workspace-analysis]] 운영 가이드 절에 박을 후보.

## 4.1 drift가 생기는 경로

| 원인 | 빈도 |
|------|------|
| 외부 도구가 `DatasetPermission` 테이블을 직접 조작 | 낮음(권장 안 함) |
| 부서 멤버 변경 후 sync 미발생(§3.4 미구현) | **중간** — 운영 중 실제 발생 |
| 코드 변경 중 sync 호출 누락 (F-5) | 낮음(테스트로 lock down) |

## 4.2 운영자 도구

| 기능 | 엔드포인트 |
|------|-----------|
| 워크스페이스 전체 drift 리스트 | `GET /workspaces/current/rbac/datasets/consistency` |
| 특정 지식 강제 재동기화 | `POST /workspaces/current/rbac/datasets/<id>/sync` |

## 4.3 drift 응답 스키마

```python
{
  "dataset_id": "...",
  "custom_visibility": "department" | "custom" | ... | None,
  "native_permission": "only_me" | "all_team_members" | "partial_members" | "",
  "custom_member_ids": ["uuid", ...],   # spx 계산 결과
  "native_member_ids": ["uuid", ...],   # 현재 Dify DB
  "drift": true | false                  # 두 집합 비교
}
```

운영자 매뉴얼 톤: "정기 점검(권장: 일 1회 배치) 시 drift=true 항목을 강제 재동기화하여 검색 권한과 권한 탭이 일치하게 유지합니다."

---

# 5. UI 차이 — K5 (UI 8종 / 유효 6종, 모달은 App과 공용)

지식 권한 탭의 UI(`web/app/components/dataset-permissions/`)는 App과 거의 같다. 차이점만 기록.

## 5.0 액션 종수 정리 (UI vs 유효)

| 구분 | 종수 | 액션 |
|------|:----:|------|
| **UI 노출** (권한 부여 모달 체크박스) | 8 | view·edit·delete·execute·publish·duplicate·manage_permission·transfer |
| **유효** (백엔드 매트릭스에 정의되어 부여 가능) | **6** | view·edit·delete·**execute**·manage_permission·transfer |
| 미정의 (UI는 보이나 부여해도 거부) | 2 | publish·duplicate |

> **execute의 의미** — 지식에서 execute는 **검색 테스트(hit-testing)** 동작. A1 §3.2 액션 정의 + `rbac_enforcement.py`의 `/hit-testing`·`/external-hit-testing` POST → execute 매핑.
>
> **publish·duplicate 미정의 근거** — A1 §C6 / §7.3 지식 매트릭스에 두 셀 없음 → INV-15 (미정의 = 모든 role 거부).

## 5.1 컴포넌트 공유 패턴

- `dataset-permissions/index.tsx` — 페이지 wrapper (App과 동형)
- `dataset-permissions/visibility-section.tsx` — 가시성 라디오 + 소유 부서 드롭다운 (App 코드와 거의 1:1)
- `dataset-permissions/acl-section.tsx` — ACL 테이블 + 권한 부여 버튼
- **권한 부여 모달은 App의 `grant-permission-modal.tsx`를 그대로 import 사용** (`acl-section.tsx` line 10)

## 5.2 액션 8종 UI 노출 / 6종 유효 — 매뉴얼 작성 시 주의

`ACTION_I18N` 매핑은 8개 액션 모두 포함하지만 코드 주석(`acl-section.tsx` line 55-57):

> `duplicate` is an app-only action today; datasets have no clone endpoint so the i18n key is only here to keep the Record exhaustive.

→ 매뉴얼 권장 처리:

- 지식 챕터의 **부여 가능 액션 표는 6개만 노출** (publish·duplicate 제외)
- "권한 부여 모달에 '발행'·'복제' 체크박스가 보일 수 있으나 지식에는 적용되지 않습니다" 한 줄 주의 — 또는 무시
- 매뉴얼 톤상 매끄러우려면 표만 6개로 두고 별도 언급 없이 진행. 부여해도 백엔드 매트릭스 미정의로 차단(`AuthorizationService.can()` line 220-221, INV-15).

## 5.3 사이드바 진입점

- 지식 상세 페이지 좌측 사이드바 — **권한** 메뉴
- 라우트: `/datasets/<datasetId>/permissions/page.tsx`
- 노출/가시성 조건은 A1 §9.2 그대로 (`permission-context.can_manage_permission` 기준 read-only 토글)

---

# 6. 원본 페이지 통합 위치 — K6 (Phase 4 본격 작성용)

[[scope-mapping]] 2026-06-02 재검토에서 **부분 수정 격상**된 Knowledge 3p. 각 페이지에 권한 섹션을 어디·어떻게 추가할지 명시.

## 6.1 `create-knowledge/introduction.mdx` — 부분 수정

- **원본 권한 언급**: 없음
- **추가 위치**: 페이지 후반 "권한 설정" H2 신설 (또는 "Settings" 인접)
- **추가 내용 1문단**:
  > "새 지식 베이스를 만들면 기본 가시성은 **비공개**입니다. 부서 단위 공유 또는 워크스페이스 전체 공개로 바꾸려면 생성 후 **권한** 탭에서 가시성 범위를 변경하시기 바랍니다. 상세 동작은 [지식 권한 설정](../permissions) 챕터를 참조하시기 바랍니다."
- **참조 링크**: [지식 권한 설정](../permissions) (새 챕터로 신설) — 본 페이지는 `knowledge/create-knowledge/introduction.mdx`이므로 `knowledge/permissions/`로 가려면 한 단계 위로(`../`)

## 6.2 `knowledge-pipeline/create-knowledge-pipeline.mdx` — 부분 수정

- **원본 권한 언급**: 없음
- **추가 위치**: 파이프라인 메타데이터 설정 섹션 또는 생성 완료 후 안내
- **추가 내용 1문단**:
  > "파이프라인을 통해 생성된 지식 베이스도 일반 지식과 동일한 권한 모델을 따릅니다. 가시성·접근 제어 변경은 [지식 권한 설정](../permissions)을 참조하시기 바랍니다."
- 파이프라인 생성 화면 자체에서 가시성을 지정할 수 있는 UI가 코드상 존재하는지 추후 확인 — 본 분석 범위 밖. **Phase 4 본격 작성 시 검증 추가** 항목.

## 6.3 `manage-knowledge/introduction.mdx` — 부분 수정 (가장 명확)

원본 line 16에 이미 권한 표 행이 있음:

```
| Permissions | Defines which workspace members can access the knowledge base.
              <Note>Members granted access to a knowledge base have all the permissions
              listed in [Manage Knowledge Content](...).</Note>|
```

- **추가 위치**: 이 행의 한국어 번역 + 행 아래에 권한 모델 4단계 + "상세는 [지식 권한 설정](../permissions) 참조" 한 줄
- **원본 Note 처리**: "권한이 부여된 멤버는 지식 콘텐츠 관리 액션을 동시에 갖는다"는 단순화된 설명. spx에서는 지식 유효 6종(조회·편집·삭제·실행·권한 관리·소유권 이전) 중 선택 부여 가능 — Note는 정확하지 않음. 한국어판은 **Note 제거 또는 정정**.
- 정정 톤:
  > "권한 부여 시 부여 대상 사용자/부서마다 조회·편집·삭제·실행·권한 관리·소유권 이전 중 필요한 액션만 선택해 줄 수 있습니다. 부여된 액션 묶음은 권한 탭에서 언제든 수정·취소할 수 있습니다."

## 6.4 신규 챕터 — 지식 권한 설정

> A1과 별도 챕터로 신설(`ko/use-spx-agent/knowledge/permissions/`). 본 분석의 결과를 토대로 1p 임시 작성 — §8에서.

---

# 7. 외부 연결 패턴 결정 적용 (2026-06-04 결정 2·4)

지식의 외부 연결 패턴 3종은 모두 6/4 회의에서 처리 확정. 본 절은 결정 결과 박제 + 본 챕터 영향 정리.

## 7.1 패턴별 결정 결과

| 패턴 | 페이지 | 6/4 초안 (컨펌 대기) | **6/4 갱신 (확정)** | 적용 결정 |
|------|--------|------------------|------------------|----------|
| **(a) 외부 데이터 import** | `sync-from-notion` / `sync-from-website` / `authorize-data-source` | 사내 정책 따라 차감 또는 유지 | **❌ 삭제 확정** | 결정 2 (outbound 외부 연결 제거) |
| **(b) 외부 KB read-only 연결** | `connect-external-knowledge-base` / `external-knowledge-api` | 사내 KB 인프라 따라 차감/유지 | **❌ 삭제 확정** | 결정 2 (outbound 외부 연결 제거) |
| **(c) 외부 노출 API (inbound)** | `manage-knowledge/maintain-dataset-via-api` | 사내 노출 정책 따라 | **✅ 유지·번역 확정** | 결정 4 (inbound 3건 유지) |

> **inbound vs outbound 구분**: (a)·(b)는 spx-agent가 외부 시스템을 호출하는 outbound 흐름이라 결정 2에 포함. (c)는 외부 시스템이 spx-agent의 지식을 호출하는 inbound라 결정 4에 포함. 결정 4 inbound 3건 중 본 문서 관련은 (c) 1건 (나머지 `publish/developing-with-apis`, `publish/webapp/embedding-in-websites`는 A4 또는 별도 산출물 범위).

## 7.2 본 챕터(`ko/use-spx-agent/knowledge/permissions/readme.mdx`) 영향

| 영향 | 처리 |
|------|------|
| **(a)·(b) 삭제로 인한 본 챕터 변경** | **없음** — 본 챕터는 가시성·ACL·동기화 모델만 다루고, 외부 연결 페이지는 인용 안 함 |
| **(c) 유지로 인한 본 챕터 추가 필요 항목** | API 키 발급의 권한 영향 — Phase 4 본격 작성 시 한 절(또는 FAQ 1줄) 추가 검토 |
| **챕터 FAQ "외부 데이터로 만든 지식도 같은 권한 모델인가?"** | (a) 삭제로 외부 데이터 import 자체가 없어짐 → **FAQ 답변 가능**: "외부 데이터 import 기능은 spx-agent에서 제공하지 않습니다. 지식은 직접 업로드 또는 파이프라인으로만 생성하며, 모두 동일한 권한 모델을 따릅니다." |

## 7.3 (c) 유지 시 추가 분석 후보 (Phase 4)

inbound API로 지식을 조작·검색할 때의 권한 영향:

- API 키는 누가 발급할 수 있나 (역할 매트릭스 + 발급 위치)
- API 키 호출이 spx 커스텀 ACL을 거치나, Dify 네이티브 권한만 보나
- 외부 시스템이 spx-agent를 호출할 때 어느 워크스페이스 멤버 권한으로 동작하나(서비스 계정 패턴)

→ 본 분석 범위 밖. Phase 4 본격 작성 시 검증 추가. 매뉴얼 본문에는 "API 호출 시에도 권한 모델이 적용됩니다" 한 줄 + 운영자 부록 또는 [[references/spx-workspace-analysis]] 인용 위임 후보.

## 7.4 scope-mapping 갱신 후보

- Knowledge 그룹 표에서 (a)·(b) 5건 ❌ 삭제 표기, (c) 1건 ✅ 유지·번역 확정 갱신 — Phase 4 본격 작성 진입 전 일괄 처리

---

# 8. 인터리브 임시 작성 — 지식 권한 챕터 1p (2026-06-04 완료)

본 분석 토대로 1p 임시 작성·등록·빌드 검증 완료.

- 작성 파일: `ko/use-spx-agent/knowledge/permissions/readme.mdx`
- 사이드바 등록: `sidebars.js` "지식" 카테고리 두 번째 항목으로 추가
- 빌드 검증: `npm run build` 성공

## 8.1 챕터 구성 (작성된 골격)

1. 소개 — A1 모델 한 줄 참조 위임 + 지식 특이사항 두 가지 명시
2. 지식 검색과 권한의 관계 — RAG 검색과 권한 탭의 일관성 자동 유지, 조회 허용만 반영 (§3.3 인용)
3. 가시성 범위별 검색 가능 멤버 표 — 4종 + 생성자 잠금 방지 (§3.2 INV-8 표면화)
4. 권한 탭 진입 — A1 §9.2 인용 위임
5. 부여 가능 액션 6종 — publish·duplicate 제외, 모달 공용으로 인한 8종 노출은 Note로 안내 (§5.2)
6. 자주 묻는 질문 4건 — 조회 미부여 / 부서 멤버 sync 미트리거 / drift 의심 / **외부 데이터 import (§7.2 결정 2 적용으로 답변 가능)**

## 8.2 작성 중 추가 확인 필요 항목 (Phase 4 본격 작성 시)

- **"운영자" 호칭 통일**: 챕터 FAQ에서 "워크스페이스 운영자에게 강제 재동기화 요청"이라고 적었으나, 사용자 관점에서 운영자가 워크스페이스 소유자/관리자인지, 별도 IT 운영자인지 모호. A3·A4와 통일된 호칭 정의 후 일괄 정리.
- **모달 공용으로 인한 복제 체크박스 노출**: 매뉴얼에 Note 한 줄 추가. 추후 UI 개선으로 액션 필터링이 들어가면 Note 제거. 백로그 후보로 [[decisions]]에 박을지 검토.
- **내부 링크 경로 형식**: 6번 작업(2026-06-04)에서 `../../app-permissions` → `../../workspace/permissions`로 갱신 완료. 이후 7번 작업(Option α)에서 챕터 본문 `[권한 설정]` 라벨로 통일. conventions.md는 절대 경로 권장이나 빌드 통과한 상태 — Phase 4 본격 작성 시 절대 경로 `/use-spx-agent/...`로 일괄 정비 후보.

## 8.3 챕터 1p 작성을 통해 보강된 본 산출물 변경 없음

본 챕터 작성 과정에서 A1처럼 산출물 내부 보강이 필요한 누락은 발견되지 않음 (모든 시나리오·동작이 본 산출물의 §3·§5에 이미 박혀 있음). 본 산출물은 사용자 매뉴얼화 준비 완료 상태로 판단.

---

# 9. A1·A3·A4가 본 문서를 인용하는 방식

| 후속 산출물 | 본 문서에서 인용할 섹션 |
|------------|----------------------|
| [[references/spx-departments-management]] (A3) | §3.4 "부서 멤버 변경 후 자동 트리거 없음" — 부서 관리 운영 가이드 시 강제 sync 안내 필요 |
| [[references/spx-workspace-analysis]] (A4) | §4 drift 검증·재동기화 운영 도구 — Workspace 설정 사이드바의 권한·감사로그 탭 운영 흐름 정리 시 |
| [[references/spx-audit-log-analysis]] (B1) | §3.4 동기화 시점 — `dataset.sync` audit log 이벤트 매핑 |

---

# 10. 후속 작업 체크리스트 (A2 마감용)

## 10.1 초안 작성 (2026-06-04)

- [x] `dataset_acl_sync_service.py` 전수 검증, 매핑/INV-8/INV-9/drift API 박제
- [x] `dataset-permissions/` UI 코드 검증, App과의 차이 식별 (모달 공용, UI 8종 노출 / 유효 6종 — publish·duplicate 매트릭스 미정의)
- [x] 원본 Knowledge 3p의 권한 언급 위치 grep — `manage-knowledge/introduction.mdx`만 명시적 행 보유, 나머지는 신규 추가
- [x] K1~K6 차이점 박제
- [x] 부분 수정 격상 3p의 통합 위치·1문단 안내
- [x] 컨펌 트랙 영향 항목 매핑 (초안)
- [x] A3·A4·B1 인용 매핑
- [x] 인터리브 챕터 1p 작성·등록·빌드 통과 (§8)

## 10.2 이사님 결정 2·4 반영 갱신 (2026-06-04, "남은 작업 3번")

- [x] frontmatter `status` 갱신·`status_history`·`related_decisions` 신설
- [x] §7 외부 연결 패턴 결정 적용으로 전면 갱신
  - §7.1 패턴 (a)/(b) 삭제 확정, (c) 유지·번역 확정 박제
  - §7.2 본 챕터 영향 처리 — (a)(b) 영향 없음, (c) Phase 4 추가 후보, FAQ "외부 데이터" 답변 가능 명시
  - §7.3 (c) 유지 시 추가 분석 후보(API 키 권한 영향)
  - §7.4 scope-mapping 갱신 후보
- [x] §8.1 챕터 골격에 FAQ 답변 가능 상태 표시
- [x] 분석본 §7.2에 FAQ 답변 근거 박제 (챕터 본문 교체는 Phase 4 본격 작성 별도 처리)

### 10.2.1 검토 결과 5건 처리 (2026-06-04 자체 검토)

> progress.md "남은 작업 3번" 라인 185-189에서 식별된 분석본 자체 수정 5건. 실질 3건 + minor 2건.

- [x] §1 "차이점 다섯 가지" → "여섯 가지" 정정 (K1~K6 = 6개와 일관)
- [x] 액션 종수 일관성 — K5 "7종" → "유효 6종 (publish·duplicate 미정의)"로 정정. §5.0 신설로 UI 8종 vs 유효 6종 구분 명시. §5.2 헤더 갱신, §6.3·§8.1 챕터 골격·§10.1 후속도 일괄 정정. **execute(검색 테스트) 적정성 확인** — A1 §3.2 + `rbac_enforcement.py` 매핑으로 정확함 박제
- [x] §6.1 상대경로 `./permissions` → `../permissions` 정정. §6.2·§6.3은 이미 `../permissions`로 일관. 3p 모두 `knowledge/<subdir>/<file>.mdx` 위치라 한 단계 위로 올라야 함
- [x] §10.2 마지막 "Phase 4 본격 작성 시 챕터 본문 FAQ 답변 교체" `[x]` → 분석본 근거 박제까지로 한정 표기. 챕터 본문 교체는 본 갱신 범위 외(별도 트랙)
- [x] §5.2 헤더 "액션 8종 노출" → "UI 8종 / 유효 6종" 부연 추가로 §1 K5와 일관

### 10.2.2 챕터 본문 액션 종수 연쇄 정정 (분석본 정정에 따른)

> 분석본 K5 정정과 일관성 유지 위해 챕터 본문도 같이 정정. KC 추상화는 별도 5번 작업이라 묶지 않음.

- [x] `knowledge/permissions/readme.mdx` 본문의 "7종 + 복제만 제외" → "6종 + publish·duplicate 제외" + UI 노출 Note 보강 (L13·L47·L58 정정 완료, 빌드 통과)

## 10.3 Phase 4 본격 작성 시 처리

- [ ] 파이프라인 생성 화면의 가시성 UI 존재 여부 추가 검증
- [ ] "운영자" 호칭 통일 — A3·A4 마감 후 일괄 정리 (현재 A3·A4 모두 완료, 통일 가능 상태)
- [ ] (c) `maintain-dataset-via-api` 본격 작성 시 API 키 권한 영향 분석 추가 — 본 갱신 §7.3 후보 검증
- [ ] scope-mapping Knowledge 그룹 표 갱신 — (a)(b) 5건 삭제·(c) 1건 유지·번역 확정 반영
- [ ] 챕터 본문 FAQ "외부 데이터로 만든 지식..." 답변 교체 — §7.2 인용
