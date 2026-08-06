	# spx-agent-docs — Fork Overrides

> 본 파일은 spx-agent용 fork 한정 컨텍스트. 루트 `CLAUDE.md`(원본 Dify 컨트리뷰터 가이드, CC BY 4.0)는 그대로 보존. 본 파일이 **원본 가이드의 일부 규칙을 덮어씁니다**.

## Fork 정체

- Upstream: `langgenius/dify-docs` @ `5c1c3a4c` (2026-03-30, "1.13.0~1.13.3 sync" 커밋)
- 매칭: **Dify 1.13.3** (spx-agent 고정 사용 버전)
- 라이선스: CC BY 4.0 — 루트 `LICENSE` + `NOTICE.md` 참조
- 히스토리: orphan — upstream sync 안 함, 단일 root 커밋

## 원본 보존 원칙 (최우선)

> **원본 파일은 최대한 손상 없이 유지**합니다. 우리 변경은 추가(`ko/` 신설, 신규 챕터 등)나 외부 파일(NOTICE.md, `.claude/CLAUDE.md`)로 표현하는 것을 기본 패턴으로 합니다.

### 보존 대상 (원칙적으로 미수정)

| 영역 | 보존 방침 |
|------|---------|
| 루트 `CLAUDE.md` | 원본 그대로 (덮어쓰는 규칙은 본 파일에서만 표현) |
| `LICENSE` | CC BY 4.0 의무, 수정 금지 |
| `en/` MDX 컨텐츠 | **수정하지 않음**. 한국어 작성은 `ko/`에 신규 작성 (워크플로 B안) |
| `writing-guides/` | 원본 가이드 그대로. 우리 규칙은 `.claude/docs/conventions.md`에서 표현 |
| `images/` 원본 자산 | 가급적 보존. spx-agent 화면 교체분은 별도 경로 또는 신규 추가 우선 검토 |
| `ja/`, `zh/` 파일 | 파일 보존 (Docusaurus 빌드 대상 아님, `ko/`만 빌드) |
| `docs.json` | **read-only 보존** (Phase 2까지 Mintlify용으로 수정, Phase 3.5 이후 동결). 네비게이션은 `sidebars.js`로 관리 |

### 신규 추가 영역 (Docusaurus 전환으로 생성)

| 영역 | 용도 |
|------|------|
| `sidebars.js` | Docusaurus 네비게이션 구조 (`docs.json` 대체) |
| `docusaurus.config.js` | Docusaurus 빌드·테마·검색 설정 |
| `package.json` | Docusaurus 의존성 관리 |
| `src/theme/MDXComponents/` | Mintlify 컴포넌트 호환 래퍼 6종 |
| `src/css/custom.css` | 테마 컬러 + 래퍼 컴포넌트 스타일 |
| `tools/migrate/` | Mintlify → Docusaurus 변환 스크립트 3종 (일회성, 참조용 보존) |

### 부득이 수정한 영역

| 영역 | 사유 | 수정 내용 |
|------|------|----------|
| `docs.json` | Phase 2까지 한국어 등록 + ja/zh nav 비활성 | 최소 수정 완료 후 **동결** (Phase 3.5 이후 read-only) |
| `.gitignore` | 옵시디언 노트 분리 필수 | `.claude/` 1줄 추가 |

### 수정 시 따를 절차

1. 수정 사유를 `.claude/docs/decisions.md`에 추가 (대안 검토 포함)
2. 원본 상태를 git 히스토리로 참조 가능하도록 유지 (`git show <commit>:<file>`)
3. 수정 범위를 최소화 (라인 단위 추가/주석 우선, 전면 재작성 회피)
4. CC BY 4.0 의무 준수: 변경 사실은 `NOTICE.md`에 명시

## 원본 가이드 덮어쓰는 규칙

원본 `CLAUDE.md`의 "Key Rules" 중 본 fork에서 **비적용 또는 변경**되는 항목:

| 원본 규칙 | 본 fork |
|----------|--------|
| "Write in English only" | ❌ **한국어로 작성·번역** (`ko/` 직접 편집) |
| "Translation sections sync automatically" | ❌ 자동 번역 파이프라인 사용 안 함, `ko/`는 수동 편집 |
| "Only edit the English section in docs.json" | ❌ `docs.json` 동결 (read-only). 네비게이션은 `sidebars.js`에서 직접 편집 |
| "Verify behavior against the Dify codebase" | ⚠️ **Dify 1.13.3** 기준 검증 (1.14 신기능 등 섞이지 않게) |

원본 가이드 중 **그대로 적용**되는 항목:
- MDX frontmatter (`title` / `description` 필수). `sidebarTitle` → `sidebar_label`로 키 변경
- writing-guides/ 참조 (formatting-guide 헤딩·리스트·코드블록 등 언어 무관 규칙)
- `.claude/skills/` auto-discover
- 커밋 컨벤션 (`{type}: {description}`)

원본 가이드 중 **비적용** (Docusaurus 전환으로 무효화):
- `mintlify dev` → `npm start` (로컬 프리뷰)
- `docs.json` nav 편집 → `sidebars.js` 편집

## 응답 규칙

- **모든 응답 한국어** (코드 식별자·주석은 영어 OK)
- 결정사항은 [[docs/decisions]]에 박기 (사유·대안·후속 트리거 포함)
- 챕터 진행 상태는 [[docs/progress]]에 즉시 반영
- 신규 용어는 [[docs/conventions]] 글로서리에 즉시 추가
- **spx-agent 코드/UI 분석 산출물은 반드시 `docs/references/spx-<주제>.md`로 박제** (휘발 금지). 분석 직후 즉시 작성, MDX 작성 시 참조
  - 기존 예: [[docs/references/spx-dashboard-analysis]]
  - 형식: 분석 범위(대상 파일·컴포넌트) → 발견(구조·동작·UI 흐름) → 챕터 작성 시 활용 포인트

## 작업 범위 (5/29 이사님 지시)

**"Use Dify" 섹션만** — 상세는 [[docs/scope-mapping]] 참조.

차감 후보 (spx-agent 미보유):
- Marketplace / Team 멤버 초대 / 외부 LLM 추가
- Getting Started / Self-host / Plugin Dev / API Reference

추가 챕터 (spx-agent 전용 신규, v2 — 2026-05-29 재구성):
- 대시보드 (KPI 카드 / 부서별 리소스 / 모델 토큰 / 드릴다운)
- 감사로그 (dify-audit 서비스, 이벤트 조회·필터·내보내기)
- 권한 설정 (가시성 4단계 / ACL / 소유권 — 앱·지식·도구 공통, 2026-06-04 rename from "앱 권한 설정")
- 사용자/부서 관리 (부서 CRUD / 멤버 배정 / KC 그룹 연동)
- ~~Keycloak SSO 로그인~~ → 삭제 (Quick Start에 흡수)
- ~~RBAC (통합)~~ → 권한 설정 + 사용자/부서 관리로 분리
- ~~설정 사이드바 (KAN-29)~~ → 삭제 (2026-06-02, Dify 원본 다이얼로그에 토글만 추가된 수준이라 별도 챕터 불필요)

## 사이드바 구조 — Option α (2026-06-04 확정)

> ⚠️ **세션 시작 시 필독**. 권위적 결정 [[docs/decisions#결정 10]] 참조. 같은 날 Option G→α로 진화한 경위 포함.

사이드바 그룹 (Workspace 흡수 + 통계·감사 별도 top-level 그룹):

```
📂 시작하기 / 노드 / 빌드 / 디버그 / 게시 / 모니터링 (평면) / 지식
📂 워크스페이스 — 기본 설정 + 앱 관리 + 사용자·부서 관리🆕 + 권한 설정🆕 + 개인 계정
📂 통계·감사 🆕 — 📁 대시보드 + 📁 감사로그
📂 튜토리얼
```

**폴더 경로 (Option α 적용)**:

| 챕터 | 경로 | 비고 |
|------|------|------|
| 권한 설정 | `ko/use-spx-agent/workspace/permissions/` | 기존 `app-permissions/`에서 이동 |
| 사용자/부서 관리 | `ko/use-spx-agent/workspace/departments/` | 기존 `workspace-management/departments/`에서 이동 |
| 워크스페이스 페이지 (개요·계정 등) | `ko/use-spx-agent/workspace/` | 기존 `workspace-management/personal-settings/` 등 이동 |
| 모니터링 | `ko/use-spx-agent/monitor/` | 평면 구조 (sub-group 없음) |
| 대시보드 | `ko/use-spx-agent/analytics-audit/dashboard/` | 기존 `dashboard/`에서 이동 |
| 감사로그 | `ko/use-spx-agent/analytics-audit/audit-log/` | 신규 (B1 인터리브 1p 작성 위치) |
| 지식 권한 | `ko/use-spx-agent/knowledge/permissions/` | 그대로 유지 |
| 지식 본체 | `ko/use-spx-agent/knowledge/` | 그대로 유지 |

**중요 주의**:
- ❌ **Option G의 `monitor/app/`, `monitor/workspace/` sub-group 구조는 폐기**. 사용 금지
- ❌ `ko/use-dify/...` 경로는 5/29 시점 안. **`ko/use-spx-agent/...`만 사용**
- ❌ `admin/` 폴더는 사용 안 함 (관리자 전용이 아니라 일반 사용자도 접근하는 페이지 포함)

## 세션 시작 시 읽을 순서

1. 루트 `CLAUDE.md` (원본 Dify 컨트리뷰터 가이드)
2. **본 파일 `.claude/CLAUDE.md`** (spx fork overrides) ← 원본의 일부 규칙 덮어씀
3. [[README]] — 옵시디언 vault 진입점 지도
4. [[docs/progress]] — 현재 작업 위치. **상단 "📍 현재 상태 한눈에" 블록부터 읽기** (전체 현황·다음 할 일 요약). 세부만 필요할 때 아래 Phase 진척 펼쳐 읽고, 과거 시간순 로그는 [[docs/progress-archive]]
5. [[docs/decisions]] — 박제된 결정사항

Phase별 추가 로드:

| Phase | 추가 로드 |
|-------|----------|
| 1 (소스 분석·범위) | [[docs/references/dify-docs-structure]] + [[docs/scope-mapping]] |
| 2 (파일럿 챕터) | [[docs/conventions]] (용어집 + MDX 규칙) |
| 4 (본격 포팅) | [[docs/scope-mapping]] + [[docs/conventions]] + [[docs/chapter-writing-checklist]] (작성 전/중/후 3단계 체크리스트) + 본 파일 §사이드바 구조 (Option α 폴더 경로) |

### 🔴 Phase 4 작업 절차 (필수)

> **기억에 의존하지 말 것.** 문서를 "읽었다"와 "따라간다"는 다르다.

체크리스트는 3단계 구조 ([[docs/chapter-writing-checklist]]):

1. **§1 그룹 사전 점검 (그룹당 1회)** — 새 그룹(예: Nodes 23p) 시작 시 scope-mapping·decisions·i18n 도메인·폴더 구조를 **한 번만** 확인. 이후 같은 그룹 페이지에서 반복 불필요.
2. **§2 페이지 작성 (매 페이지)** — 원본 읽기→전역 규칙 스캔→i18n 라벨 추출→본문 작성→**직역체 3패스**(grep + 소리 내 읽기 + 원문 대조)→i18n 사후 검증→전역 규칙 grep. 참조 문서(`[[conventions]]`, `[[translationese-guide]]`)는 "이미 알고 있으니까"로 건너뛰지 말고 **열어놓고** 대조.
3. **§3 그룹 마감 (그룹 완료 시 1회)** — 빌드 최종 검증, progress 메타 갱신, 글로서리 일괄 확인, deferred 정리.

부분 수정(§A)·신규 챕터(§B)·데이터 시각화(§C)는 해당 유형에만 추가 적용.

## `.claude/` 폴더 정체

- **Junction → 옵시디언 vault**: `C:\Users\Administrator\Documents\Obsidian\Daily\3. 프로젝트\spx-agent-docs`
- 루트 `.gitignore`에 등록 (git 미추적)
- 옵시디언에서 편집 = VS Code에서 편집 = 같은 파일
- 내부: `CLAUDE.md`(본 파일) / `README.md` / `docs/` (메모) / `skills/` (Dify 보존)

## 수정 금지·주의

- **루트 `CLAUDE.md`** — 원본 Dify 컨트리뷰터 가이드. **수정 금지** (덮어쓰는 규칙은 본 파일로만 표현)
- **`LICENSE`** — CC BY 4.0 의무, 수정 금지
- **`NOTICE.md`** — 출처 표기 (내용 갱신 OK, 라이선스·baseline 정보 제거 금지)
- **`.claude/skills/`** — Dify 원본 스킬, 직접 수정 X
- **`ja/`, `zh/`** — 자동 번역 결과물. 한국어판만 만들 거라 **전체 삭제 후보** (Phase 1 결정)
- **upstream sync 시도 금지** — orphan 채택. cherry-pick만 예외 허용

## 외부 리소스

- **upstream 리모트**: `https://github.com/langgenius/dify-docs.git` (정보 참조용, push 금지)
- ~~**GitHub origin**: `mjlee-spelix/spx-agent-demo`~~ → **리모트 제거 완료** (Phase 3.5, Mintlify Cloud 폐기)
- **사내 git**: Phase 6 사내 nginx 배포와 함께 결정 (현재 미연결)
- **호스팅**: **Docusaurus + 사내 nginx** 확정 (2026-06-02, [[docs/decisions]])

## 빌드 도구 (Docusaurus, 2026-06-02 전환)

- **Phase 0~3**: Mintlify CLI (`mintlify dev`) — 베이스라인 + 파일럿 작성
- **Phase 3.5 이후**: **Docusaurus**로 전환 — 사내망/외부 GitHub 의존 회피, 자체 호스팅, 비용 0

### 개발 명령어

| 명령어 | 용도 |
|--------|------|
| `npm start` | 로컬 개발 서버 (`http://localhost:3000`) |
| `npm run build` | 프로덕션 빌드 → `build/` 정적 산출물 |
| `npm run serve` | 빌드 결과 로컬 서빙 (배포 전 확인) |

### 주요 설정 파일

| 파일 | 역할 |
|------|------|
| `docusaurus.config.js` | 사이트 설정, 테마, 검색, i18n |
| `sidebars.js` | 네비게이션 구조 (Mintlify `docs.json` 대체) |
| `src/theme/MDXComponents/` | Mintlify 컴포넌트 호환 래퍼 6종 |
| `src/css/custom.css` | 테마 컬러 + 래퍼 스타일 |

### 전환 원칙

- `docs.json`은 read-only 보존 (NOTICE 차원), `sidebars.js`가 실제 네비게이션
- Mintlify 컴포넌트(`<Info>`, `<Frame>` 등)는 MDXComponents 글로벌 래퍼로 호환 — MDX 본문 수정 없이 동일 태그 사용
- 검색: `@easyops-cn/docusaurus-search-local` (한국어, 오프라인)
- 상세 작업 계획: [[docs/references/mintlify-to-docusaurus]]

## 흔한 실수 → 가드

| 실수 | 가드 |
|------|------|
| 옵시디언 노트를 git에 커밋 | `.gitignore`에 `.claude/` 등록 |
| 용어 일관성 깨짐 | [[docs/conventions]] 글로서리 즉시 참조·갱신 |
| 1.14 신기능 docs 섞임 | baseline `5c1c3a4c` 시점 기준만 |
| 원본 CLAUDE.md 수정 | 본 파일에서만 override |
| `.claude/CLAUDE.md` 자동 로드 가정 | Claude Code 자동 로드 불확실 — 세션 시작 시 명시 로드 권장 |
