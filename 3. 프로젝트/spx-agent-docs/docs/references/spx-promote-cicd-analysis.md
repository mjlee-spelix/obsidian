---
title: 워크플로 배포(Promote / CI/CD) 신규 챕터 — promote 모듈 분석
phase: Phase 4 / 신규 챕터 (CI/CD)
status: 1차 분석 완료 (2026-06-18) — 컨펌 대기
audience: spx-agent 사용자 매뉴얼 작성자 (CI/CD 챕터 집필자)
purpose: |
  spx-agent의 "워크플로 배포(Promote)" 기능을 사용자 매뉴얼 관점으로 정리.
  dev Dify 에서 만든 워크플로(App)를 Git 을 거쳐 prod Dify 로 승격하는 흐름이며,
  spx-agent 내부의 신규 Promote 모듈(web + api)로 구현되어 있다.
  본 문서는 ①사용자가 보는 화면(Settings → 배포)·②동작 흐름·③권한·④노출 조건(dev 전용)을
  박제하고, 백엔드 구조는 "사용자에게 안 보이는 영역"으로 분리 표기한다.
  → 2차(mdx) 작성 시 본 문서의 §2·§6(사용자 노출)만 본문화하고, §5(백엔드)는 제외한다.
source_root: C:\Users\Administrator\Projects\spx-agent
source_plan_doc: C:\Users\Administrator\Projects\spx-agent\docs\harness\promote-cicd.md
code_verified_at: 2026-06-18
---

# 1. 본 문서의 위치

> ⚠️ **본 분석본은 구조·동작·권한 박제용(정전)입니다. 본문(mdx)은 이 구조를 그대로 옮기지 마시기 바랍니다.** 2차 작업(mdx)은 **사용자가 알아야 하는 것만** 쓰고(백엔드 과정 제거), **"개발 환경에만 있는 기능"임을 콜아웃으로 명시**해야 합니다(§7 작성 지침).

본 문서는 **사용자 매뉴얼 집필을 위한 분석 노트**다. "워크플로 배포(Promote)"는 spx-agent 본체(Dify fork) 안에 추가된 **신규 모듈**으로, dev Dify 에서 작성·게시한 워크플로(App)를 **Git 저장소를 거쳐 prod Dify 로 안전하게 승격**한다. Dify 원본에는 없는 spx-agent 전용 기능이다.

본 문서는 다음을 정전화한다:

1. **노출 조건** — 왜 일부 환경에서만 보이는가 (feature flag, §3) ← **콜아웃 근거**
2. **화면·탭·모달** — 사용자가 실제로 보는 UI (§2·§4) ← **mdx 본문 대상**
3. **두 가지 핵심 동작** — Snapshot / Promote 흐름 (§4) ← **mdx 본문 대상**
4. **권한** — 누가 Snapshot 하고 누가 Promote 하나 (§6) ← **mdx 본문 대상**
5. **백엔드 구조** — Git/정규화/overlay/prod import (§5) ← **mdx 제외(참고용)**

> **출처**: 설계 플랜 `docs/harness/promote-cicd.md` (spx-agent repo) + 실제 코드 검증(2026-06-18). 코드와 플랜이 다른 부분은 코드를 정전으로 삼고 본문에 표시했다.

> **인접 분석**: 배포 인프라(Jenkins/태그 배포)는 [[references/jenkins-deploy-194]]·[[references/deployment-config-extracts]]. 단, 그쪽은 **spx-agent 코드/이미지 배포**용이고 본 문서는 **워크플로(App 정의) 배포**용으로 영역이 다르다(§5.5).

---

# 2. 한눈에 — 사용자가 보는 것 (mdx 본문 핵심)

```
[설정(Settings) 모달] ── 좌측 메뉴 "배포" (감사 로그 바로 위, owner/admin + 활성화 시에만 노출)
   │
   ├─ Hero: "워크플로 배포 (Promote)"
   ├─ 환경 카드: dify-dev (현재) / dify-prod (운영)
   └─ 가로 탭 3개
        ├─ 워크플로 : dev 앱 카드 그리드 + [Snapshot 만들기] / [운영 반영] 버튼 + 모달
        ├─ 설정     : Git/Prod 연결/권한 정책/토큰 카테고리/Jenkins (읽기 전용)
        └─ 활동     : 최근 snapshot/promote 이력 10건

[스튜디오 앱 목록] ── 앱 카드에 작은 뱃지
   ├─ "snapshot 필요" (노랑) — dev 가 git 보다 앞섬
   └─ "promote 대기" (파랑)  — git 이 prod 보다 앞섬
   └─ 뱃지 클릭 → 설정 모달의 "배포" 탭 자동 열림
```

사용자 입장 요약: **별도 페이지/메뉴가 아니라 "설정 → 배포" 탭 하나**에서 모든 흐름이 끝난다. 그리고 이 메뉴는 **개발 환경에서만 보인다**(§3).

---

# 3. 노출 조건 — "개발 환경에만 있다" (콜아웃 근거)

> 🔴 **2차 mdx 작성 시 가장 중요한 부분.** 이 기능은 **운영(prod) 사용자에게는 메뉴 자체가 안 보인다.** 따라서 mdx 본문 상단에 "이 기능은 개발 환경에서만 제공됩니다" 류의 콜아웃을 반드시 넣어야 한다.

## 3.1 런타임 환경변수 토글 `PROMOTE_ENABLED`

빌드 분리가 아니라 **런타임 환경변수 한 줄**로 켜고 끈다. dev/prod 이미지는 동일하다.

| 항목 | 값 | 근거 (코드 검증) |
|------|-----|----------------|
| 환경변수 | `PROMOTE_ENABLED` | `api/configs/feature/__init__.py` L1407-1410 |
| **기본값** | **`False`** (꺼짐) | 같은 곳. **명시적으로 켜야만 동작** |
| dev 설정 | `.env.dev` 에 `PROMOTE_ENABLED=true` | 플랜 §9.4 |
| prod 설정 | prod 도 `PROMOTE_ENABLED=true` 가능하나, prod 에서 "배포 메뉴 자체"의 의미가 없음(승격 대상이 자기 자신) — 실사용은 dev 인스턴스 | 플랜 §4·§9.4 |

> 정확히는 "dev 에서만 코드가 빠진" 것이 아니라 **"환경변수로 꺼져 있으면 안 보이는"** 구조다. prod 이미지에도 코드는 들어 있지만 라우트가 비활성(외부 호출 시 404)이고 메뉴도 안 뜬다. **사용자 관점에서는 "개발(dev) 환경의 설정에서만 보이는 기능"으로 설명하면 충분하다.**

## 3.2 2중 게이트 (백엔드 + 프론트)

| 계층 | 동작 | 코드 |
|------|------|------|
| 백엔드 라우트 | `PROMOTE_ENABLED` False 면 컨트롤러 자체를 import 안 함 → 모든 `/promote/*` 라우트 404 | `api/controllers/console/promote/__init__.py` L1-14 (`if dify_config.PROMOTE_ENABLED:`) |
| 시스템 기능 플래그 | `SystemFeatures.promote_enabled` 로 프론트에 전달 (기본 false) | `web/types/feature.ts` L67, L110 / `services/feature_service.py` |
| 프론트 메뉴 | `isCurrentWorkspaceManager && promoteEnabled` 일 때만 "배포" 메뉴 push | `web/app/components/header/account-setting/index.tsx` L128-135 |
| 스튜디오 뱃지 | `promoteEnabled` false 면 뱃지 컴포넌트가 `null` 반환 | `web/app/components/apps/promote-changed-badge.tsx` L30 |

→ 즉 **활성화(dev) + owner/admin** 두 조건을 모두 만족해야 메뉴가 보인다. (권한은 §6)

---

# 4. 화면 구성 — 사용자가 보는 것 (mdx 본문 대상)

## 4.1 진입 — 설정 → 배포

| 항목 | 내용 | 코드 |
|------|------|------|
| 진입점 | 우측 상단 계정 아이콘 → 설정(Settings) → 좌측 메뉴 **배포** | `account-setting/constants.ts` L12 (`DEPLOY: 'deploy'`) |
| 위치 | 좌측 메뉴에서 **감사 로그 바로 위** | `account-setting/index.tsx` L128-135 (배포) vs L137-144 (감사 로그) |
| 아이콘 | 로켓 (`i-ri-rocket-2-line` / 활성 `i-ri-rocket-2-fill`) | 같은 곳 |
| 라벨 | `t('settings.deploy')` → 한국어 **"배포"** | i18n (§8 주의 — 일부 하드코딩) |
| 노출 조건 | owner/admin + `promoteEnabled` (§3) | 같은 곳 |

## 4.2 배포 페이지 상단 — Hero + 환경 카드

`account-setting/deploy-page/index.tsx`

| 영역 | 표시 내용 |
|------|----------|
| **Hero** | 제목 "**워크플로 배포 (Promote)**" + 설명 "dev 에서 작성한 워크플로를 Git 저장소를 거쳐 prod 로 안전하게 승격합니다." (L32-45) |
| **환경 카드** (`<EnvSection />`) | **dify-dev**(현재 인스턴스, 항상 정상) / **dify-prod**(운영, 연결 설정 여부에 따라 상태 표시) 2개 카드 (sections.tsx L42-63) |

## 4.3 가로 탭 3개

`deploy-page/index.tsx` L49-70. 탭 키/라벨은 코드에 하드코딩(L23-25).

| 탭 | 컴포넌트 | 사용자가 보는 것 |
|----|---------|----------------|
| **워크플로** | `WorkflowsSection` | dev 앱 카드 그리드(중간 화면 2열·큰 화면 3열). 각 카드에 상태 뱃지 + dev/git/prod 시각 + 액션 버튼. (sections.tsx L81-136) |
| **설정** | `SettingsSection` | **읽기 전용** 설정 6블록 (§4.6) (L140-201) |
| **활동** | `ActivitySection` | 최근 10건 이력 테이블: 액션 뱃지 / 앱 slug / git SHA / 행위자·시각 (L232-271) |

## 4.4 워크플로 탭 — 카드 상태와 버튼 (핵심)

`promote/app-card.tsx`. 카드마다 **상태에 따라 뱃지와 버튼이 달라진다.** 이것이 사용자가 가장 자주 보는 화면.

| 상태 | 뱃지 | 버튼 | 의미 (사용자 언어) |
|------|------|------|------------------|
| 동기화됨(inSync) | — | **"반영됨"** (비활성, 체크) | dev = git = prod, 더 할 것 없음 |
| dev 가 앞섬(needsSnapshot) | **"snapshot 필요"** / 미등록 시 "stage 변경" | **"Snapshot 만들기"** (주황) | dev 에서 수정·게시했고 아직 Git 에 안 올림 |
| 게시 필요(publishRequired) | **"게시 필요"** | "게시 필요" (비활성) | dev 에서 초안만 있고 publish 안 함 → 먼저 게시해야 함 |
| git 이 앞섬(gitAhead) | **"promote 대기"** | **"운영 반영"** (파랑) | Git 에는 올라갔는데 prod 에 아직 미반영 |

> 카드 상태 근거: app-card.tsx L38-55(뱃지), L73-107(버튼). 시각 표시 3종(dev_updated_at / git_committed_at / prod_synced_at)은 `web/service/promote.ts` 의 `PromoteApp` 타입.

## 4.5 두 가지 핵심 동작 — Snapshot / Promote 모달

### 4.5.1 Snapshot (dev → Git) — `promote/snapshot-modal.tsx`

"Snapshot 만들기" 클릭 시 모달. 상태: `idle → running → (done | no_change | fail)` (L17)

| 단계 | 사용자가 보는 것 |
|------|----------------|
| idle | 설명 + (최초 등록 시) "overlay.prod.yaml 이 자동 생성됨" 안내 |
| running | 로더 + "내보내기 → 정규화 → Git 커밋/푸시 중..." |
| done | 체크 + slug·sha·commit_sha 표시 |
| **no_change** | "dev 게시본과 Git 스냅샷이 동일 → 갱신 생략" (변경 없음) |
| fail | 에러 메시지 |

> 사용자 메시지로 풀면: **"Snapshot = 지금 dev 에 게시된 워크플로를 Git 에 한 벌 저장(스냅샷)"**. 변경이 없으면 아무것도 안 한다.

### 4.5.2 Promote (Git → prod) — `promote/promote-modal.tsx`

"운영 반영" 클릭 시 모달. 상태: `idle → running → (done | fail)`. 모달 제목 "운영 반영 — {앱 이름}".

| 단계 | 사용자가 보는 것 |
|------|----------------|
| idle | "검증 → overlay 적용 → prod 가져오기 → 확인 순서로 진행" |
| running | 로더 |
| done | 체크 + import_status·prod_sha·prod_app_id |
| fail | **에러 코드별 안내 메시지**(FAIL_CODE_HINTS, L38-49) |

**실패(차단) 사유 — 사용자가 마주칠 수 있는 것** (`PromoteApiFailCode`, service/promote.ts):

| 코드 | 사용자 의미 |
|------|-----------|
| `prod_not_configured` | 운영(prod) 연결이 설정 안 됨 |
| `not_registered` / `no_snapshot` | 먼저 Snapshot 부터 해야 함 |
| **`overlay_missing`** | **환경값(토큰)이 빠짐** → 누락 키를 `${CATEGORY:key}` 형식으로 목록 표시 (promote-modal.tsx L146-160) |
| `dify_version_mismatch` | dev/prod Dify 버전 불일치 |
| `prod_drift` | 운영에서 누군가 직접 수정함(drift 감지) → 차단 |
| `prod_import_failed` | 운영 가져오기 실패 |
| `locked` | 같은 앱을 누군가 동시에 승격 중 |

> 사용자 메시지로 풀면: **"운영 반영 = Git 에 저장된 스냅샷을 운영(prod) Dify 로 가져오기"**. 환경값이 빠졌거나 운영이 변경됐으면 안전하게 막는다.

## 4.6 설정 탭 — 읽기 전용 6블록 (`sections.tsx` L140-201)

모두 **조회만**(편집 불가). mdx 에서는 간단히 "현재 연결·정책을 확인하는 화면" 정도로.

1. **Git 저장소** — Remote URL / 기본 브랜치 / 로컬 클론 / 커밋 작성자 / HEAD SHA
2. **Prod Dify 연결** — Base URL / 인증 방식 / API Key(마스킹) / Keycloak client / 필요 역할 / 연결 상태
3. **권한 정책** — Snapshot(Editor 이상) / Promote(Owner·Admin) / 동시성 락
4. **토큰 치환 카테고리** — 뱃지 4종: **KB / TOOL / WEBHOOK / MODEL**
5. **Jenkins 연동** — 활성 여부·URL·Job (현재 미사용/예정)
6. **알림 채널** — Slack·이메일 (계획)

## 4.7 스튜디오 앱 카드 뱃지 — `apps/promote-changed-badge.tsx`

스튜디오(앱 목록)의 각 앱 카드에 작은 뱃지가 붙어 "여기 배포할 게 있다"를 알린다.

| 조건 | 뱃지 | 색 |
|------|------|-----|
| `dev_ahead` | "snapshot 필요" | 노랑(테두리) |
| `git_ahead` | "promote 대기" | 파랑 |
| 둘 다 아님 | (없음) | — |

- 클릭 → `setShowAccountSettingModal({ payload: DEPLOY })` → **설정 모달의 배포 탭이 자동으로 열림** (L48-51)
- `promoteEnabled` false 면 렌더링 안 함 (L30)

---

# 5. 백엔드 구조 — 사용자에게 안 보이는 영역 (mdx 제외)

> 🔴 **2차 mdx 작성 시 본 절은 본문에 넣지 않는다.** "백엔드 과정 제거" 요구사항 대상. 참고용으로만 박제.

## 5.1 컨트롤러 4종 (`api/controllers/console/promote/`)

| 엔드포인트 | 클래스 | 역할 | 파일 |
|-----------|--------|------|------|
| `GET /console/api/promote/apps` | `PromoteAppsApi` | dev 앱 목록 + git 상태 비교(dev_ahead/git_ahead) + 활동 10건 | apps.py L123-202 |
| `POST /console/api/promote/apps/<id>/snapshot` | `PromoteSnapshotApi` | dev export → 정규화 → git commit/push → DB row upsert | snapshot.py L43-180 |
| `POST /console/api/promote/apps/<id>/promote` | `PromoteRunApi` | 5단계 검증 → overlay → prod import → confirm | promote.py L47-194 |
| `GET /console/api/promote/config` | `PromoteConfigApi` | 설정 탭용 read-only 조회(민감정보 마스킹) | config.py L41-76 |

## 5.2 서비스 (`api/services/promote/`)

| 파일 | 역할 |
|------|------|
| `git_repo.py` | spx-workflows repo 관리 (ensure_repo / read_app / write_app / commit / push / compute_hash / head_sha) — subprocess |
| `normalize.py` | Dify export YAML 정규화 + 토큰화. `TOKEN_CATEGORIES = ("KB","TOOL","WEBHOOK","MODEL")`, 휘발성 키 제거 |
| `overlay.py` | `${CATEGORY:key}` 치환 + 누락 토큰 검출 (정규식 `\$\{(KB\|TOOL\|WEBHOOK\|MODEL):([^}]+)\}`) |
| `dify_remote_client.py` | prod Dify console API wrapper (import_app / confirm_import / export_app, httpx + Bearer) |
| `concurrency.py` | `spx_promote_lock` row lock, TTL 1시간, 같은 앱 동시 promote 차단 |

## 5.3 promote 5단계 검증 (promote.py 내부)

1. prod 연결 설정 확인 (L59-82)
2. DB row + last_dev_sha 존재 확인 (L59-82)
3. git 산출물 `apps/<slug>/app.yaml` 읽기 (L85-101)
4. overlay 적용 + 누락 토큰 검증 (L104-111)
5. `meta.dify_version` vs `CURRENT_DSL_VERSION` 버전 검증 (L114-121)
6. (prod_app_id·last_prod_sha 둘 다 있을 때만) prod export hash 비교 = **drift 감지** (L128-146)

## 5.4 DB 테이블 3종 (`api/models/promote.py`)

| 테이블 | 용도 |
|--------|------|
| `spx_promote_app` | 앱별 dev/git/prod SHA·메타 (PK=slug) |
| `spx_promote_activity` | 활동 로그 (snapshot/promote/rollback/register) |
| `spx_promote_lock` | 동시성 락 (PK=app_slug) |

마이그레이션: `api/migrations/versions/...add_promote_tables.py`

## 5.5 인프라 — Jenkins 와의 관계

- **워크플로 CI/CD ≠ 코드 배포 CI/CD.** 본 모듈은 워크플로(App 정의)를 옮기고, [[references/jenkins-deploy-194]] 의 Jenkins 파이프라인은 spx-agent 컨테이너 이미지를 배포한다.
- 현재 promote 는 backend 가 prod Dify console API 를 **직접 호출**하므로 Jenkins 없이 동작. Jenkins `promote-workflow` job 은 플랜상 **보류(Phase 4)**.

---

# 6. 권한 — 누가 Snapshot 하고 누가 Promote 하나 (mdx 본문 대상)

두 동작의 권한이 **다르다**. 운영 영향이 큰 Promote 는 더 높은 권한이 필요하다.

| 동작 | 필요 권한 | 데코레이터 (코드) | 이유 |
|------|----------|------------------|------|
| **Snapshot** (dev → Git) | **Editor 이상** | `@edit_permission_required` (snapshot.py L50) | dev 게시 권한자 = dev 변경 권한자 |
| **Promote** (Git → prod) | **Owner / Admin** | `@is_admin_or_owner_required` (promote.py L54) | 운영 영향 큰 결정, 게이트 필요 |
| 목록·설정 조회 | 인증된 사용자 | (제한 없음) — 단 메뉴 자체가 owner/admin 게이트 | apps.py / config.py |
| 배포 메뉴 노출 | owner/admin | `isCurrentWorkspaceManager` (§3.2) | |

> 사용자 메시지: **"워크플로를 Git 에 스냅샷하는 것은 편집자 이상, 운영에 반영하는 것은 관리자/소유자만 가능합니다."**

---

# 7. 2차 작업(mdx) 작성 지침

> 2차는 1차 컨펌 후 진행. 아래는 본 분석본 → mdx 변환 시 지침 박제.

## 7.1 반드시 지킬 것 (사용자 요구사항)

1. **사용자가 알아야 하는 것만** 쓴다 — §2·§3·§4·§6 중심.
2. **백엔드 과정 제거** — §5(컨트롤러·서비스·DB·5단계 내부 검증·정규화 알고리즘)는 본문에 넣지 않는다. "내부적으로 검증 후 안전하게 반영" 수준으로만.
3. **개발 환경 전용 콜아웃** — 본문 상단(또는 소개 직후)에 `<Info>` 또는 `<Note>` 콜아웃으로 "이 기능은 **개발 환경**에서만 제공되며, 워크스페이스 관리자(소유자/관리자)에게만 표시됩니다" 명시 (§3 근거).

## 7.2 콜아웃 톤 예시 (그대로 옮겨 쓸 수 있음)

> 💡 **개발 환경 전용 기능**
> 워크플로 배포(Promote)는 **개발 환경의 설정 화면에서만** 제공됩니다. 또한 워크스페이스 **소유자·관리자**에게만 메뉴가 표시됩니다. 일반 사용자나 운영 환경에서는 이 메뉴가 보이지 않습니다.

## 7.3 본문 구성(안)

```
# 워크플로 배포 (Promote)
<Info> 개발 환경 전용 콜아웃 </Info>   ← §7.2

## 배포란?            ← dev→Git→prod 한 문장 + 그림(§2)
## 배포 화면 열기      ← 설정 → 배포 (§4.1)
## 워크플로 탭        ← 카드 상태/뱃지 표 (§4.4)
## 워크플로를 Git 에 저장하기 (Snapshot)  ← §4.5.1 (사용자 동작 위주)
## 운영에 반영하기 (Promote)             ← §4.5.2 + 차단 사유(§4.5 표) 사용자판
## 권한                ← §6 한 문단
## 스튜디오 뱃지        ← §4.7
(설정·활동 탭은 짧게 또는 생략 판단)
```

## 7.4 폴더·사이드바 위치 (결정 필요 — 컨펌 사항)

- 본 챕터는 Dify 원본에 없는 **spx-agent 신규 챕터**. Option α 사이드바 구조([[../CLAUDE]] §사이드바)에 **새 슬롯이 필요**하다.
- 후보: **워크스페이스 그룹** 내 (감사로그·대시보드처럼 관리 기능) 또는 **통계·감사** 옆 신규 그룹.
- 폴더 후보: `ko/use-spx-agent/workspace/deploy/` 또는 `ko/use-spx-agent/promote/`.
- → **2차 진입 전 사용자/이사님 컨펌 필요** (사이드바 그룹 결정은 [[../decisions]] 박제 대상).

---

# 8. 코드 검증 발견 사항 (작성 시 주의)

| # | 발견 | 영향 |
|---|------|------|
| 1 | **i18n 라벨 대부분 하드코딩** — 탭명("워크플로/설정/활동"), Hero, 모달 문구, 설정 블록 제목 등이 i18n json 이 아닌 컴포넌트 코드에 한국어로 직접 박혀 있음. 전용 `promote/deploy` i18n 파일 없음 | mdx 표기는 **코드의 하드코딩 한국어 문자열**을 정전으로 사용. 단 메뉴 라벨 "배포"는 `t('settings.deploy')` 사용 — `common` 네임스페이스 등록 여부 확인 필요(미확인 시 영어 fallback 가능성) |
| 2 | `PROMOTE_ENABLED` 기본값 **False** | "개발 환경 전용" 콜아웃의 직접 근거 (§3) |
| 3 | 설정·활동 탭은 read-only / 미래 기능(Jenkins·알림) 포함 | mdx 에서 "예정"·"읽기 전용"임을 흐리지 말 것. 과한 약속 금지 |
| 4 | 플랜 문서(`promote-cicd.md`)의 PR 흐름·Gitolite·SSH 등은 **운영 인프라 영역**(Phase 5, 미완) | 사용자 매뉴얼 범위 밖. mdx 제외 |

---

# 9. 한국어 표기 박제 (글로서리 반영 후보)

| 화면 표기(코드) | 본문 표기 |
|----------------|----------|
| 배포 / Promote | 워크플로 배포(Promote) |
| Snapshot 만들기 | (워크플로를) Git 에 저장 / 스냅샷 |
| 운영 반영 | 운영(prod)에 반영 / 승격 |
| snapshot 필요 / promote 대기 | 스냅샷 필요 / 운영 반영 대기 |
| stage 변경 / 게시 필요 / 반영됨 | (그대로) |
| dify-dev / dify-prod | 개발 환경 / 운영 환경 |
| overlay (환경값) | 환경별 설정값 |
| token (KB/TOOL/WEBHOOK/MODEL) | 환경 치환 값 |
| drift | (운영) 직접 변경 감지 |

> ⚠️ 본문에서는 "dev/prod" 를 "개발/운영"으로 풀어쓰되, 화면에 "dify-dev" 라고 표기되는 카드는 화면 라벨 존중. "Keycloak" 직접 노출은 전역 규칙 #6(외부 시스템 추상화) 적용 — 설정 탭의 Keycloak client 항목 언급 시 주의(애초에 §4.6 설정 탭은 비중 축소 권장).

---

# 10. 후속 체크리스트

## 10.1 1차 (본 분석, 2026-06-18)
- [x] promote-cicd.md 플랜 정독 + 실제 코드 검증 (feature flag·UI·모달·권한·백엔드·i18n)
- [x] 사용자 노출(§2·§4·§6) vs 백엔드(§5) 분리 박제
- [x] "개발 환경 전용" 콜아웃 근거(§3) + mdx 작성 지침(§7) 박제
- [x] 코드 검증 발견 4건(§8) + 한국어 표기(§9)
- [ ] **사용자 컨펌 대기** — 본 분석본 + 사이드바 위치(§7.4)

## 10.2 2차 (mdx — 컨펌 후)
- [ ] 사이드바 그룹/폴더 확정 ([[../decisions]] 박제) + `sidebars.js` 등록
- [ ] 본문 작성 (§7.3 구성) — 백엔드 제외, 개발 환경 콜아웃 포함
- [ ] i18n 라벨 사후 검증(§8-1) + 글로서리 갱신(§9)
- [ ] 빌드 검증 + [[../chapter-writing-checklist]] 통과
