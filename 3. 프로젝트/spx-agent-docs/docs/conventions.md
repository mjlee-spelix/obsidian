# 작성 컨벤션

> 한국어 문서 작성 기준. Phase 2 파일럿 검증 완료 (2026-05-29).
> 원본 `writing-guides/`와의 관계: [[decisions]] "writing-guides 처리 — 하이브리드" 참조.

> **Phase 4 본문 작성 시 [[chapter-writing-checklist]] 참조** — 작성 전/중/후 3단계 체크리스트 (전역 규칙·MDX·문체·UI 라벨 검증·빌드 검증·후속 박제).

## MDX 포맷 규칙

> **빌드 도구가 Mintlify → Docusaurus로 전환되었습니다** (Phase 3.5, 2026-06-02).
> 기존 Mintlify 전용 JSX 컴포넌트(`<Info>`, `<Frame>`, `<Steps>` 등)는 **MDXComponents 글로벌 래퍼**(`src/theme/MDXComponents/`)로 동일 태그명을 유지합니다. MDX 본문에서의 사용법은 동일하며, 구현만 Docusaurus 네이티브로 교체되었습니다.
>
> 원본 `writing-guides/formatting-guide.md`의 **헤딩·리스트·코드 블록·테이블** 등 언어 무관 규칙은 그대로 적용됩니다.

아래는 자주 참조할 항목 요약:

- **frontmatter**: `title` 필수. `description`은 원본 `en/` 페이지에 있는 경우에만 동일하게 작성 (없으면 생략). `sidebar_label`로 사이드바 표시명 지정 가능 (Mintlify의 `sidebarTitle` 대체).
- **헤딩**: H2 → H3 → H4 순서 (건너뛰기 금지). 한국어 헤딩은 Title Case 불필요 (한국어에 해당 없음).
- **Bold**: UI 요소(버튼·메뉴·탭·필드명)와 핵심 용어 첫 등장에 사용.
- **리스트**: `-` 사용 (순서 있으면 `1.` `2.`). 항목 간 빈 줄 하나.
- **코드 블록**: 언어 태그 필수 (```python, ```bash 등).
- **이미지**: `<Frame>` 컴포넌트 + `caption` + `alt` 속성 (MDXComponents 래퍼로 동작).
- **내부 링크 — 마크다운 vs `<Card href>` 컨벤션 (2026-06-04 C1 작업 중 박제)**:
  - **마크다운 링크** `[텍스트](경로)`: **상대경로 사용** (`./subpage`, `../sibling`). Docusaurus가 빌드 시 검증(broken link 검사)함. `/ko/` prefix 불필요. 앵커 참조 포함 가능
  - **`<Card href="...">` 등 raw `<a>` 렌더 컴포넌트**: ⚠️ **절대경로 의무** (`/use-spx-agent/...`)
    - 사유 1 — `Card.jsx`(MDXComponents 래퍼)가 raw `<a>` 태그를 렌더 → **Docusaurus broken link 검사 우회**. "빌드 통과 = 링크 정상"이 보장 안 됨
    - 사유 2 — `trailingSlash` 미설정 환경에서 상대경로(`./`·`../`)가 서빙 URL에 따라 깨질 수 있음
    - 적용: `<Card href="/use-spx-agent/workspace/permissions/" ...>` ✅ / `<Card href="../permissions/" ...>` ❌
  - **요약**: 마크다운 링크는 상대경로, raw `<a>` 컴포넌트는 절대경로. 헷갈리면 절대경로가 안전 (검증 우회 방지)
  - **타겟 미생성(dangling) 시에도 상대경로 유지** — 빌드 시 broken link 경고가 뜨지만 `onBrokenLinks: 'warn'`이라 빌드는 통과. 타겟 페이지 작성 시 자동 해소됨. 절대경로로 우회하면 나중에 소급 정정해야 하므로 처음부터 상대경로로 작성
  - **`readme.mdx`/`index.mdx`는 폴더 슬러그** (2026-06-10 박제): Docusaurus가 폴더 인덱스로 처리해 URL이 폴더 경로가 됨. 링크 시 `/readme`·`/index`를 붙이지 말 것 — `./import-text-data/` ✅ / `./import-text-data/readme` ❌ (**타겟이 존재해도 broken**). create-knowledge 1차 배치에서 `/readme` 링크가 실제 broken으로 확인됨
  - **앵커(anchor) — ko 헤딩 기준** (2026-06-10 박제): ko 페이지의 앵커는 한글 헤딩에서 자동 생성됨 (예: `## 검색 설정` → `#검색-설정`). 원본 영문 앵커(`#setting-the-retrieval-setting`)는 ko에서 무효. 타겟 ko 페이지가 미생성일 때는 앵커 없이 페이지 링크만 걸고, 타겟 작성 시 한글 앵커로 보완
- **컴포넌트** (MDXComponents 래퍼 제공, import 불필요):
  - 어드모니션: `<Info>`, `<Tip>`, `<Note>`, `<Warning>`, `<Check>`, `<Callout>`
  - 레이아웃: `<Frame>`, `<Steps>`/`<Step>`, `<Tabs>`/`<Tab>`, `<Card>`/`<CardGroup>`, `<Accordion>`/`<AccordionGroup>`
- **간격**: 컴포넌트·코드블록·헤딩 전후 빈 줄 1개. 이중 빈 줄 금지.
- **네비게이션**: 페이지 등록은 `sidebars.js`에서 관리 (Mintlify의 `docs.json` 대체). `docs.json`은 원본 보존(read-only).

상세는 원본 파일을 직접 참조: `writing-guides/formatting-guide.md`

## 한국어 문체·톤 (확정)

> 원본 `writing-guides/style-guide.md`는 영어 문체 기준이라 한국어에 직접 적용 불가.
> 아래는 한국어 문서용으로 재정의한 규칙.

### 기본 원칙

- **자연스러운 한국어 우선**: 직역 X (5/29 이사님 지시)
- **문장 길이**: 영어 원문 1문장 ↔ 한국어 1~2문장 (자연스러운 분리 허용)
- **간결함**: 모든 문장이 가치를 더해야 함. 불필요한 말 제거하되, 가독성 희생하지 않음

### 톤 일관성 (번역·신규 챕터 공통, 2026-06-05 확정)

- **모든 `ko/` 챕터는 동일한 문체·톤·어미**를 따른다 — 번역 챕터든, 원문 없는 spx-native 신규 챕터(권한 설정·사용자/부서 관리·대시보드·감사로그·지식 권한)든 구분 없음.
- **신규 챕터는 "직역 검토"만 면제**될 뿐(EN 원문이 없어 직역이 발생할 수 없음 — [[translationese-guide]] 검토 이력 3차 참조), **톤·구조·용어·콜아웃 일관성은 동일하게 적용**한다. 오히려 기준점(원문)이 없어 톤이 튀기 쉬우므로 더 주의할 것.
- **작성 기준점(reference)**: 신규 챕터를 쓸 때는 같은 사이드바 그룹의 기존(번역) 챕터를 한 편 펼쳐 두고 **어미·콜아웃 밀도·헤딩 패턴**을 맞춘다. 새로 톤을 만들지 말고 기존 톤에 합류시킬 것.

### 존댓말 정책 — 합쇼체 (확정, 1차 파일럿 검증 완료)

- **합쇼체** 사용: `~할 수 있습니다`, `~됩니다`, `~합니다`
- 선택적 동작: `~할 수 있습니다`
- 필수 동작: `~해야 합니다`
- **청유·명령(사용자에게 행동을 안내하는 문장) 표준 어미: `~하시기 바랍니다`** (2026-06-05 확정)
  - `참고하세요` ❌ → `참고하시기 바랍니다` ✅ / `요청하십시오`·`새로고침하십시오` → `요청하시기 바랍니다`·`새로고침하시기 바랍니다`
  - `~하세요`(너무 가벼움)·단독 `~하십시오`(번역 챕터와 톤 불일치)는 지양하고 **`~하시기 바랍니다`로 통일**
  - 단, 단순 평서·기능 설명(`~합니다`, `~할 수 있습니다`)은 그대로 — 표준 어미는 *행동 안내 문장*에만 적용

### 영문 병기 정책 (확정)

- 전문 용어 첫 등장 시 `한국어(English)` 병기, 이후 한국어만
- UI 요소는 한국어 UI가 있으면 한국어, 없으면 영어 그대로 bold
- 페이지 frontmatter: `title`만 필수, `description`은 불필요하면 생략, `sidebar_label`로 사이드바 표시명 지정 가능

### 언어 무관 스타일 패턴 (style-guide에서 차용)

아래 패턴은 원본 style-guide의 영어 패턴을 한국어에 맞게 적용:

- **위치 먼저 → 동작 나중**: "**설정** 패널에서 토글을 활성화합니다." (위치 → 동작)
- **사용자 성과 중심**: 기술 메커니즘 대신 사용자가 달성하는 것 설명
- **문제 → 해결 구조**: 기능 소개 시 해결하려는 문제부터, 그다음 해결책
- **점진적 공개**: 핵심 먼저, 세부사항은 필요에 따라 추가
- **의사결정 정보 제공**: 특정 설정 강요 대신 시나리오와 트레이드오프 제시

### 콜아웃 사용 규칙 (style-guide에서 차용)

| 컴포넌트 | 용도 | 예시 |
|----------|------|------|
| `<Info>` | 일반 참고 정보, 버전·배포 관련 맥락 | "이 기능은 spx-agent 1.13.3부터 지원됩니다." |
| `<Tip>` | 유용한 제안·단축키 | "Ctrl+K로 빠르게 검색할 수 있습니다." |
| `<Note>` | 놓치면 문제가 될 수 있는 중요 정보 | "모델 제공자를 먼저 설정해야 LLM 노드를 사용할 수 있습니다." |
| `<Warning>` | 오류·데이터 손실 가능 동작 | "이 작업은 되돌릴 수 없습니다." |

- **남용 금지**: 콜아웃이 많으면 중요도가 희석됨. 정말 중요한 정보에만 사용
- **제한 위치**: 사용자가 행동 전에 알아야 할 제한사항은 섹션 **시작 부분**에 배치

### 피해야 할 패턴

- **과도한 불릿**: 연결되는 내용을 불릿으로 쪼개지 말 것. 문단으로 작성
- **UI 반복 설명**: 사용자가 화면에서 직접 볼 수 있는 내용(기본값, 레이블)은 반복하지 않음. 화면에 없는 맥락·이유를 제공
- **기능 중심 도입**: "이 기능은 ~를 할 수 있게 해줍니다" → "~할 수 있습니다" 또는 명령형
- **군더더기 표현**: "~하기 위해서는", "참고로", "유의할 점은" 등 제거
- **없는 기능 안내**: "~는 제품 기능이 아닙니다. 운영 차원에서 별도로 대응하시기 바랍니다" 류. 제품에 없는 기능을 언급하며 대안을 권고하지 않는다. 해당 문장 자체를 삭제
- **유효 정보 동반 삭제**: SaaS 맥락(#1·#8)을 덜어낼 때 제품에 실재하는 정보(토큰 소비·기본값·동작 제약)까지 함께 지우지 말 것. 상업 맥락만 덜고 나머지는 재서술 (scope-mapping #8 부분 제거 원칙). 예: "토큰을 소비합니다. 자세한 건 pricing page 참조" → "추가 토큰이 소비됩니다"는 보존, "pricing page" 부분만 제거

### 검수 발견 전역 규칙 (G1~G7) — 2026-06-15 워크스페이스 검수 박제

> 워크스페이스 그룹 검수에서 추출. **워크스페이스 한정이 아니라 모든 그룹 작성·검수 시 공통 적용**. 출처: [[review/08-워크스페이스]] §1, [[1. Daily/2026-06-15]].

| 규칙 | 내용 | 연계 |
|------|------|------|
| **G1** 없는 기능을 "없다"고 설명하지 말 것 | 제품에 없는 기능(워크스페이스 전환·멀티 워크스페이스 메뉴·부서 삭제 UI 등)은 "제공되지 않습니다"라고 명시하지 말고 **언급 자체를 안 함** | §피해야 할 패턴 "없는 기능 안내" |
| **G2** UI 라벨 = 실제 한국어 화면 텍스트 그대로 | 영문·임의 번역 금지, i18n/화면 대조 필수 (예: "Edit info"→"정보 편집하기", "Export DSL"→"DSL 내보내기", "가시성 범위"→"공개 범위", "취소 버튼"→"삭제 버튼") | §UI 라벨 검증 절차 |
| **G3** 개발자 시점 용어 → 사용자 표현 | '모달'·'다이얼로그'·'확인 다이얼로그'·'현재 화면에서는' 회피 → 동작·결과로 서술("현재는 ~") | 신규 |
| **G4** 전문 용어는 첫 등장 시 정의 | DSL 등은 처음 나올 때 풀어쓰기. 설명이 뒤 문단에 나오면 안 됨 | 신규 |
| **G5** 사용자가 안 궁금한 내부 동작·군더더기 제거 | = 전역 규칙 #8 문장 적합성 (예: "외부 시스템 그룹 동기화"·"여러 그룹이 부서로 매핑"·"비공개에선 소유 부서 표시 안 함") | scope-mapping #8 |
| **G6** 주어·주체 모호한 문장 회피 | "자동 부여되는 권한" — 누가 부여하는지 명시 | 신규 |
| **G7** 어색한 직역 헤딩·용어 다듬기 | "## 진입" 등 직역 헤딩 자연스럽게 | §직역체 회피 |

**그룹 규칙 (워크스페이스 한정 — 사실·구조, W1~W5)**: 메인 메뉴는 화면 **상단**(좌측 아님), 순서 대시보드/탐색/스튜디오/지식/도구(W1) · 대시보드는 메인 메뉴에만·역할 무관(W2) · 워크스페이스 설정 진입=우측 상단 아바타→설정(W3) · 권한 챕터 사이드바 탭 이름='개요'(W4) · 개인 설정=언어·시간대만, 로그인 방식 설명 X(W5).

### 데이터 조회·시각화 챕터 서술 원칙 (대시보드·감사로그·모니터링)

> 5/28 이사님 지시("그림 화려하게보다 의미 설명에 초점, 로그 항목들이 어떤 건지 설명, 어떻게 업무에 적용·활용하는지")를 본문 작성 원칙으로 확장. 출처: [[1. Daily/2026-05-28]] 회의 메모 / [[references/ibm-research]].
> **적용 대상**: 화면을 "보는" 챕터 — 대시보드, 감사로그, 모니터링(앱별 통계 = Analysis·Logs 한정). 단 Annotation Reply는 설정 기능이라 비대상.

**핵심: 차트·카드·컬럼을 나열·정의하는 데서 멈추지 말 것.** 각 요소마다 아래 3단을 답한다 (모두 한 문장씩이라도):

1. **무엇을 보여주는가 (의미)** — 이 차트/카드/컬럼이 나타내는 지표가 무엇인지
2. **어떤 질문에 답하나 / 왜 중요한가** — 사용자가 이걸 읽으면 무엇을 알 수 있는지
3. **업무에 어떻게 활용하나** — 어떤 판단·행동으로 이어지는지 (가능하면 짧은 시나리오)

**서술 형태**:
- 표로 라벨만 나열하는 구조 < 각 요소를 "지표 → 읽는 법 → 활용" 흐름의 문단/짧은 설명으로
- 화면에 그대로 보이는 라벨·기본값·수치는 반복하지 않음 (§피해야 할 패턴과 동일). 화면에 **안 보이는** "왜 중요한지·어떻게 읽는지"를 제공
- 이미지·캡처는 보조. 캡션은 "화면 의미·업무 활용 초점" (§이미지 처리)

**대시보드 특화**:
- KPI 카드 4종·차트 2종·부서별 활동 테이블·드릴다운 각각 위 3단 적용
- 증감 배지(▲녹색/▼빨강)가 의미하는 것 = 기간 대비 추세 → 어떤 행동 신호인지
- 부서별/모델별 비교가 운영자에게 주는 인사이트 (자원 편중·비용 집중 파악 등)

**감사로그 특화**:
- 각 로그 항목(시각·행위자·액션·대상 등 컬럼)이 무엇을 뜻하는지 설명
- 액션 라벨 각각이 어떤 사용자 행위인지 (글로서리 §감사 로그 — 카테고리·행위자·액션 라벨 연계)
- 어떤 상황에 조회·필터·내보내기 하는지 (감사 추적·보안 점검 시나리오)

### 직역체 회피 (translationese)

> "직역 X"의 *방법*. 선언만으로는 직역이 안 잡혀서, 절차·마커를 명문화함.
> **Before/After 사례집은 [[translationese-guide]] 참조** (마커별 교정 예시 모음).

#### 번역 절차 — 의미 재구성 (직역을 끊는 핵심)

1. **문단 단위로 의미만 파악** — 문장을 1:1로 옮기지 말 것 (영어 골격이 그대로 남는 원인)
2. **원문을 보지 않고 한국어로 새로 작성** — "원문 가리고 다시 쓰기"
3. **한국어 본문만 소리 내 읽고** 어색하면 아래 마커표로 교정
4. **마지막에만 원문과 대조** — 정보·표·콜아웃 누락 여부만 확인

#### 직역체 마커 체크리스트

| 직역 신호 | 원인(영어 구조) | 교정 방향 |
|:--|:--|:--|
| `~을 위한 N` (만들기 위한 플랫폼) | `for` / to-부정사 | `~하는 N` |
| 무생물 주어 (이 기능은 ~제공/결정합니다) | `X provides/determines` | 사용자·행위 주어 또는 피동 |
| 명사 `of`/명사화 체인 (~의 ~의, 권한 부여의 단위) | noun-of-noun | 주어·서술어(동사)로 풀기 |
| `~ 위에서/위에 동작` (엔진 위에서) | `on top of` | `~으로` |
| `~로부터 생성` (파일로부터) | `from` | `~로 만들다` |
| `~를 통해` 남용 (API를 통해) | `through/via` | `~로` 또는 생략 |
| 괄호 한자어 동격 (~없습니다(상호 배타적)) | appositive 형용사 | 문장으로 풀어쓰기 |
| 피동 남용 (~되어집니다, ~에 의해) | 영어 수동태 | 능동 |
| 군더더기 (한 번에 일괄, 참고로, ~하기 위해서는) | redundancy | 삭제 |
| 영어 어순·직역 어휘 (서식 있는 텍스트, ~를 참조해) | lexical / word order | 한국어 관용 표현·어순 재배치 |
| `~할 수 있게 해주는 N` (통합할 수 있게 해주는 기능) | `allows/lets you to` | `~하는 N` |

## 이미지 처리

- 원본 Dify 이미지 → spx-agent 화면으로 **Phase 5에서 교체**
- Phase 2~4: 텍스트 중심 작성, 이미지는 placeholder 또는 원본 참조
- `<Frame>` 컴포넌트 사용 + `caption`(화면 의미·업무 활용 초점) + `alt` 속성
- 5/28 이사님 지시: "화려한 그림보다 의미·업무 활용 초점"

## 절차 시각화 (5/28 이사님 지시)

- 지식/에이전트 생성 등 절차는 `<Steps>` 컴포넌트로 시각화 (MDXComponents 래퍼 제공)
- Mermaid 다이어그램: Docusaurus에서는 `@docusaurus/theme-mermaid` 플러그인 추가 필요 (현재 미설치, 필요 시 추가)

## 용어집 (Glossary)

> 신규 용어 등장 시 즉시 본 표에 추가. 원본 `writing-guides/glossary.md`의 영문 용어를 참조용으로 대조.

### UI 라벨 검증 절차 (2026-06-04 박제) ⭐ 챕터 작성 전 필수

**문제**: 표준 한국어 번역과 spx-agent UI 실제 표기가 다를 수 있음 (예: "변수 인스펙터" vs UI 실제 라벨 "변수 검사")

**해결**: spx-agent 코드의 **i18n 파일이 권위 출처**. UI에 노출되는 라벨은 모두 여기서 추출.

> ⚠️ **예외 — UI 문구가 틀린 경우**: i18n/화면 라벨이 stale·오역·미등록·키 재활용이라 **실제 동작·명칭과 어긋나는** 케이스가 있다. 이때는 UI를 따르지 말고 코드 동작/제품 표준 명칭으로 작성한다. 알려진 목록: [[references/ui-source-discrepancies]] (예: 부서 모달 "한 부서에만"=stale, 감사로그 "데이터셋"=구 명칭). 라벨 검증 시 본 카탈로그를 먼저 대조.

**검증 범위** (2026-06-09 확장): **UI 라벨**(버튼·탭·컬럼명)뿐 아니라 **도메인 개념어**(해당 기능의 핵심 동사·상태값·수치 단위)도 i18n에서 검색합니다. 예: 어노테이션의 hit→"조회", match→"일치", threshold→"임계값". UI에 직접 보이지 않더라도 i18n에 정의된 번역이 있으면 그것이 권위입니다.

#### 권위 우선순위 (충돌 해결 규칙)

| 출처 | 권위 | 적용 |
|------|------|------|
| **1. spx-agent i18n 파일** (`web/i18n/ko-KR/*.json`) | 최상위 | UI 실제 노출 라벨. 사용자가 화면에서 보는 것 |
| **2. dify-audit `audit-meta.ts`** (별도 앱) | 최상위 | 감사 로그 도메인 한정 |
| **3. spx-agent 자체 추가 용어 (코드에 한국어 키 없음)** | 글로서리 박제로 권위화 | 부서·가시성 4단계·신규 챕터 라벨 등 |
| **4. 본 글로서리** | 2차 | i18n 검증 결과를 기록·재사용하는 사전. **i18n과 충돌하면 i18n 승** |
| **5. `writing-guides/glossary.md` 표준 번역** | 참조용 | Dify 원본 가이드 — i18n과 충돌 시 i18n 승 |

**충돌 시나리오 처리**:

| 상황 | 처리 |
|------|------|
| i18n과 글로서리가 일치 | 그대로 |
| i18n과 글로서리가 충돌 | **i18n 라벨로 글로서리 갱신**. 기존 라벨은 `~~strikethrough~~`로 보존 (변경 이력) |
| i18n에 라벨 있음 + 글로서리에 없음 | 글로서리에 즉시 추가, `i18n 출처` 컬럼에 파일·키 박제 |
| i18n에 없음 + spx-agent 자체 추가 용어 | 글로서리에서 권위 (`spx-agent 전용` 섹션) |
| 둘 다 없음 | i18n 우선 확인 → 없으면 표준 번역 채택, 후속 검증 deferred |

> ⚠️ **기존 글로서리 항목 다수는 i18n 검증 미완**. 노드 23종·UI 레이블 등은 `writing-guides/glossary.md` 표준 번역에서 차용 — 챕터 작성 시 해당 도메인 i18n 재검증 권장.
>
> 이미 i18n·코드 검증된 항목: A1 권한 모델·A4 로그인 옵션·B1 감사 로그 라벨 (분석본 작성 시 코드 검증됨).

#### i18n 파일 위치

```
C:\Users\Administrator\Projects\spx-agent\web\i18n\ko-KR\*.json
```

#### 챕터 도메인 → i18n 파일 매핑

| 챕터 도메인 | 우선 검색 파일 | 보조 검색 |
|----------|---------|---------|
| Get Started | `common.json`, `login.json`, `layout.json` | `app.json` |
| Nodes (워크플로 노드) | `workflow.json` | `tools.json` (Tools 노드) |
| Build (워크플로 편집) | `workflow.json` | — |
| Debug (워크플로 디버그) | `workflow.json` (`debug.*` 키) | — |
| Publish (게시·WebApp) | `share.json`, `app.json` | `app-api.json` (API 게시) |
| Monitor (앱 모니터링) | `app-debug.json`, `app-log.json`, `app-annotation.json` | — |
| Knowledge | `dataset.json`, `dataset-creation.json`, `dataset-documents.json`, `dataset-hit-testing.json`, `dataset-settings.json`, `pipeline.json`, `dataset-pipeline.json` | — |
| Workspace (워크스페이스 설정) | `common.json`, `layout.json`, `oauth.json` | `register.json` |
| 권한 설정 (신규) | `common.json` (또는 `app.json` 권한 키), spx-agent 자체 추가 키 | — |
| 사용자/부서 관리 (신규) | `layout.json`, `common.json` (멤버·부서), spx-agent 자체 추가 키 | — |
| 대시보드 (신규) | `layout.json` (사이드바), spx-agent 자체 추가 키 | — |
| 감사로그 (신규) | dify-audit `src/lib/audit-meta.ts` (별도 앱) | — |

#### 검증 워크플로 (챕터 작성 전 매번)

1. **도메인 식별** — 본 챕터가 어느 i18n 파일에 매핑되는지 위 표 확인
2. **권위 라벨 grep** — 챕터에 등장할 키워드로 i18n 파일 검색
   ```bash
   # 예: Debug 챕터 작성 전
   grep -i "stepRun\|단계 실행" web/i18n/ko-KR/workflow.json
   grep -i "variableInspect\|변수" web/i18n/ko-KR/workflow.json
   grep -i "runHistory\|기록" web/i18n/ko-KR/workflow.json
   ```
3. **권위 라벨로 글로서리 갱신** — 발견된 UI 라벨이 본 글로서리에 없으면 즉시 추가 (UI 검증 표기 명시)
4. **챕터 본문 작성·검증** — 글로서리 + i18n 검증 결과 기반으로 작성

#### 검증 실패 패턴 (피해야 할 케이스)

| 패턴 | 사례 |
|------|------|
| 표준 번역 임의 적용 | "Variable Inspect" → "변수 인스펙터" ❌ (UI는 "변수 검사") |
| 영어 그대로 음역 | "Step Run" → "스텝 런" ❌ (UI 확인 필요) |
| 글로서리 미경유 작성 | 매번 즉흥 번역 → 일관성 깨짐 |

#### 작성 후 검증 (선택)

작성된 챕터 본문에서 UI 라벨 후보를 grep해서 권위 라벨과 비교:
```bash
# 예: debug 챕터 작성 후 검증
grep -hE "변수 [^,。!？\.]+" ko/use-spx-agent/debug/*.mdx | sort -u
# → 본 결과를 i18n grep 결과와 비교
```

### 핵심 개념

| English | 한국어 | 원본 glossary 대조 | 비고 |
|---------|--------|-------------------|------|
| Workflow | 워크플로우 | ✅ 일치 (工作流) | |
| Chatflow | 채팅 플로우 | ⚠️ i18n 교정 (`app.json` `types.advanced`) | ~~챗플로우~~ → 채팅 플로우 (i18n 권위) |
| Agent | 에이전트 | ⚠️ 원본은 Agent 유지 | 한국어판은 "에이전트" 사용 |
| Text Generator | 텍스트 생성기 | ✅ (文本生成应用) | |
| knowledge base | 지식 | ⚠️ 원본은 "knowledge base" 소문자 | 한국어에서 "지식"으로 통일, "지식베이스" X |
| plugin | 플러그인 | ✅ (插件) | spx-agent 미사용 → 챕터 삭제 후보 |
| Dify tool / Tool | 도구 | ✅ (工具) | |
| workspace | 워크스페이스 | ✅ (工作区) | |
| WebApp | 웹앱 | ✅ (WebApp) | |
| App | 앱 | | |
| Multimodal | 멀티모달 | | 첫 등장 시 병기 |
| Streaming | 스트리밍 | | 첫 등장 시 병기 |

### 모델 관련

| English | 한국어 | 비고 |
|---------|--------|------|
| model | 모델 | |
| model provider | 모델 제공자 | |
| LLM | LLM | 약어 유지 |
| embedding model | 임베딩 모델 | |
| rerank model | 리랭크 모델 | |
| token | 토큰 | |
| prompt | 프롬프트 | |
| system instruction | 시스템 지시문 | |

### 워크플로우 노드 (i18n `workflow.json` 검증 완료, 2026-06-08)

| English (UI) | 한국어 | i18n 블록명 | 비고 |
|-------------|--------|-----------|------|
| User Input | 시작 | 시작 (`blocks.start`) | 캔버스 노드명·변수 프리픽스 모두 "시작". "사용자 입력"은 노드 추가 시 노드 피커에서만 노출 (2026-06-15 실제 UI 확인). 문서도 "시작" 사용 |
| LLM | LLM | LLM | |
| Knowledge Retrieval | 지식 검색 | 지식 검색 | |
| Answer | 답변 | 답변 | 채팅 플로우 종단 노드 |
| Output | 출력 | 출력 (`blocks.end`) | 워크플로우 종단 노드 |
| Agent | 에이전트 | 에이전트 | 노드 (앱 타입과 구분) |
| Question Classifier | 질문 분류기 | 질문 분류기 | |
| IF/ELSE | 조건 분기 | IF/ELSE | UI는 영어 유지. 문서 제목·본문은 "조건 분기" 사용 |
| Code | 코드 실행 | 코드 | UI "코드" + 문서에서 "실행" 보충 |
| Template | 템플릿 | 템플릿 | `blocks.template-transform`="템플릿". 문서명도 "템플릿"으로 통일 (2026-06-15 결정, 기존 "템플릿 변환" 폐기). Jinja2 |
| HTTP Request | HTTP 요청 | HTTP 요청 | |
| Variable Aggregator | 변수 집계자 | 변수 집계자 | ~~변수 집약기~~ i18n 교정 |
| Variable Assigner | 변수 할당자 | 변수 할당자 | ~~변수 할당기~~ i18n 교정 |
| Iteration | 반복 | 반복 | |
| Loop | 루프 | 루프 | |
| Parameter Extractor | 매개변수 추출기 | 매개변수 추출기 | ~~파라미터 추출기~~ i18n 교정 |
| Doc Extractor | Doc 추출기 | Doc 추출기 | ~~문서 추출기~~ i18n 교정 |
| List Operator | List 연산자 | List 연산자 | ~~리스트 처리~~ i18n 교정 |
| Human Input | 사람 입력 | 사람 입력 | ~~사람 개입~~ i18n 교정. HITL |
| Schedule Trigger | 일정 트리거 | 일정 트리거 | ~~예약 트리거~~ i18n 교정 |
| Webhook Trigger | 웹훅 트리거 | 웹훅 트리거 | |
| Plugin Trigger | 플러그인 트리거 | 플러그인 트리거 | 삭제 대상 (결정 1) |
| Tool | 도구 | 도구 | |

### 노드 구성 필드·동작 라벨 (i18n 사후 검증, 2026-06-08)

> 배치 2·3 사후 검증에서 발견한 UI 필드·동작 라벨. 문서 작성 시 이 라벨을 사용할 것.

| English (UI) | 한국어 | i18n 키 | 적용 노드 |
|-------------|--------|---------|----------|
| Query Text | 질의 텍스트 | `nodes.knowledgeRetrieval.queryText` | 지식 검색 |
| Query Images | 이미지 조회 | `nodes.knowledgeRetrieval.queryAttachment` | 지식 검색 |
| User Actions | 사용자 작업 | `nodes.humanInput.userActions.title` | 사람 입력 |
| Timeout | 시간 초과 | `nodes.humanInput.timeout.title` | 사람 입력 |
| Max Loop Count | 최대 루프 수 | `nodes.loop.loopMaxCount` | 루프 |
| Extend (array op) | 연장 | `nodes.assigner.operations.extend` | 변수 할당자 |
| Remove First / Remove Last | 첫 번째 제거 / 마지막 제거 | `nodes.assigner.operations.remove-first/last` | 변수 할당자 |
| Write Mode | 쓰기 모드 | `nodes.assigner.writeMode` | 변수 할당자 |

### 지식 & 검색

| English | 한국어 | 비고 |
|---------|--------|------|
| chunk | 청크 | "세그먼트" X |
| chunking | 청킹 | |
| retrieval | 검색 | |
| indexing | 인덱싱 | |
| embedding | 임베딩 | |
| metadata | 메타데이터 | |
| Vector Search | 벡터 검색 | |
| Full-Text Search | 전체 텍스트 검색 | i18n `retrieval.full_text_search.title`. ~~전문 검색~~ 교정 (2026-06-10) |
| Hybrid Search | 하이브리드 검색 | |
| Top K | Top K | 영어 유지 |
| score threshold | 점수 임계값 | |
| Retrieval Testing | 검색 테스트 | i18n `dataset-hit-testing.title` |
| Records | 레코드 | i18n `dataset-hit-testing.records`. 검색 테스트 이벤트 로그 |
| Source Text | 소스 텍스트 | i18n `dataset-hit-testing.input.title`. 검색 테스트 입력 필드 |
| Index Method | 인덱스 방법 | i18n `dataset-settings.form.indexMethod` |
| Retrieval Method | 검색 방법 | i18n `dataset-settings.form.retrievalSetting.method` |
| Delimiter | 구분 기호 | i18n 툴팁 기준. UI 라벨 "세그먼트 식별자"와 불일치 (2026-06-10) |
| Maximum chunk length | 최대 청크 길이 | |
| Chunk Overlap | 청크 중첩 | i18n `dataset-creation.stepTwo.overlap` |
| General (chunk mode) | 일반 | i18n `dataset-creation.stepTwo.general` |
| Parent-child (chunk mode) | 부모-자식 | i18n `dataset-creation.stepTwo.parentChild` = "부모 - 자식" (공백 차이, 문서는 "부모-자식" 사용) |
| Rerank Model | 재순위 모델 | i18n `form.retrievalSetting.multiModalTip` 기준 |
| High Quality (index) | 고품질 | i18n `dataset-settings.form.indexMethodHighQuality` |
| Economical (index) | 경제적 | i18n `dataset-settings.form.indexMethodEconomy` |
| Inverted Index | 역인덱스 | i18n `retrieval.invertedIndex.title` |
| Q&A Mode | Q&A 모드 | 영문 유지 |
| Knowledge Pipeline | 지식 파이프라인 | i18n `pipeline.json` |
| Blank Knowledge Pipeline | 빈 지식 파이프라인 | i18n `dataset-pipeline.json` `creation.createFromScratch.title` |
| Orchestration (pipeline) | 오케스트레이션 | i18n `pipeline.json` `publishToast.desc` |
| Data Source | 데이터 소스 | i18n `dataset-pipeline.json` `addDocuments.steps.chooseDatasource` |
| Input Field (pipeline) | 입력 필드 | i18n `dataset-pipeline.json` `inputField` |
| User Input Field | 사용자 입력 필드 | i18n `dataset-pipeline.json` `inputFieldPanel.title` |
| Global Inputs | 전역 입력 | i18n `dataset-pipeline.json` `inputFieldPanel.globalInputs.title` |
| Unique Inputs | 고유한 입력 | i18n `dataset-pipeline.json` `inputFieldPanel.uniqueInputs.title` |
| Import from DSL File | DSL 파일에서 가져오기 | i18n `dataset-pipeline.json` `creation.importDSL` |
| Publish as Knowledge Pipeline | 지식 파이프라인으로 게시 | i18n `pipeline.json` `common.publishAs` |
| Archive (document) | 아카이브 | i18n `dataset-documents.json` `list.action.archive` |
| Unarchive (document) | 아카이브 해제 | i18n `dataset-documents.json` `list.action.unarchive` |
| Generate Summary | 요약 생성 | i18n `dataset-documents.json` `list.action.summary` |
| Rename (document) | 이름 바꾸기 | i18n `dataset-documents.json` `list.table.rename` |
| Permissions (knowledge) | 권한 | i18n `dataset-settings.json` `form.permissions` |
| Only Me (permission) | 나만 | i18n `dataset-settings.json` `form.permissionsOnlyMe` |
| Weighted Score | 가중 점수 | 리랭크 설정 |
| Multi-path Retrieval | 다중 경로 검색 | |

### UI 레이블 (사이드바·탭)

| English (UI) | 한국어 | 비고 |
|-------------|--------|------|
| Studio | 스튜디오 | 메인 내비 `menus.apps` |
| Knowledge | 지식 | 메인 내비 `menus.datasets` |
| Explore | 탐색 | 메인 내비 `menus.explore` |
| Plugins | 플러그인 | 사이드바 메뉴 (spx-agent 미사용) |
| Tools | 도구 | 메인 내비 `menus.tools` |
| Dashboard (nav) | 대시보드 | 메인 내비 `menus.dashboard`. 관리자 전용 |
| Settings → Model Providers | 모델 제공자 | `settings.provider` |
| Settings → Members | 사용자 관리 | `settings.members` |
| Settings → Departments | 부서 관리 | `settings.departments` |
| Department → Manage Members | 구성원 관리 | `rbac.department.manageMembers`. **부서 스코프는 "구성원"**(현재 구성원·추가 가능·구성원 수=`memberCount`). 워크스페이스 탭 "사용자 관리"(`settings.members`)와 구분. 단 "## 멤버 페이지" 등 일반 서술 헤딩은 구조 변경 risk로 "멤버" 유지 (2026-06-15) |
| Settings → Language | 언어 | `settings.language` |
| Settings → My Account | 내 계정 | `settings.account` |
| Settings group: Workspace | 작업 공간 | `settings.workplaceGroup`. 문서 개념 "워크스페이스"와 차이 (2026-06-10) |
| Orchestrate | 오케스트레이트 | 앱 상세 탭. ~~편성~~ → i18n `common.appMenus.promptEng` 교정 (2026-06-09) |
| Monitoring | 모니터링 | 앱 상세 탭. Dify docs의 페이지 제목 "Analysis"도 동일 영역 → 한국어 번역 시 페이지 제목 **"모니터링"**으로 통일 (UI label 우선) |
| API Access | API 액세스 | 앱 상세 탭. ~~API 접근~~ → i18n `common.appMenus.apiAccess` 교정 (2026-06-09) |
| Logs & Annotations | 로그 & 주석 | 앱 상세 탭 |
| Annotation Reply (feature toggle) | 주석 응답 | `app-debug.json` `feature.annotation.title`. 앱 설정 "기능 추가" 패널의 토글 라벨 |
| Annotation Reply (management page) | 어노테이션 답변 | `app-annotation.json` `name`. 어노테이션 관리 화면 제목 |
| Add Features (button) | 기능 추가 | `app-debug.json` `operation.addFeature`. 오케스트레이트 화면의 버튼 |
| Score Threshold (annotation) | 점수 임계값 | `app-debug.json` `feature.annotation.scoreThreshold.title`. ~~유사도 임계값~~ i18n 교정 (2026-06-09) |
| Hit (annotation) | 조회 | `app-annotation.json` `viewModal.hit`. ~~적중~~ i18n 교정 (2026-06-09) |
| Hit History | 조회 기록 | `app-annotation.json` `viewModal.hitHistory`. ~~적중 이력~~ i18n 교정 |
| Hits (count) | 조회수 | `app-annotation.json` `table.header.hits` |
| Match (annotation) | 일치 | `app-annotation.json` `hitHistoryTable.match`. ~~매칭~~ i18n 교정 (2026-06-09) |
| Accurate Match | 정확한 일치 | `app-debug.json` `feature.annotation.scoreThreshold.accurateMatch` |
| Easy Match | 간단한 일치 | `app-debug.json` `feature.annotation.scoreThreshold.easyMatch` |
| Publish | 게시 | |
| Preview | 미리보기 | |
| Test Run | 테스트 실행 | |

### 로그 (i18n `app-log.json` 검증, 2026-06-09)

| English (UI) | 한국어 | i18n 키 | 비고 |
|-------------|--------|---------|------|
| Workflow Log (title) | 워크플로우 로그 | `app-log.json` `workflowTitle` | 워크플로우 앱 전용 로그 화면 |
| Triggered From (column) | 트리거 기준 | `app-log.json` `table.header.triggered_from` | 워크플로우 로그 컬럼 |
| Trigger: Web App | 웹앱 | `app-log.json` `triggerBy.appRun` | |
| Trigger: Debugging | 디버깅 | `app-log.json` `triggerBy.debugging` | |
| Trigger: Webhook | 웹훅 | `app-log.json` `triggerBy.webhook` | |
| Trigger: Schedule | 일정 | `app-log.json` `triggerBy.schedule` | |
| Log & Annotation (tab) | 로그 및 어노테이션 | `common.json` `appMenus.logAndAnn` | 앱 상세 탭 |

### 모니터링 지표 (i18n `app-overview.json` 검증, 2026-06-09)

| English | 한국어 | i18n 키 | 비고 |
|---------|--------|---------|------|
| Total Messages | 총 메시지 수 | `analysis.totalMessages.title` | |
| Total Conversations | 총 대화 수 | `analysis.totalConversations.title` | |
| Active Users | 활성 사용자 수 | `analysis.activeUsers.title` | |
| Avg User Interactions | 평균 사용자 상호작용 수 | `analysis.avgUserInteractions.title` | |
| Avg Session Interactions | 평균 세션 상호작용 수 | `analysis.avgSessionInteractions.title` | |
| Avg Response Time | 평균 응답 시간 | `analysis.avgResponseTime.title` | |
| Token Output Speed | 토큰 출력 속도 | `analysis.tps.title` | |
| Token Usage | 토큰 사용량 | `analysis.tokenUsage.title` | |
| User Satisfaction Rate | 사용자 만족도율 | `analysis.userSatisfactionRate.title` | |

### 디버그 (i18n 검증 완료, 2026-06-04)

| English (UI) | 한국어 | i18n 키 | 비고 |
|-------------|--------|---------|------|
| Variable Inspector | 변수 검사 | `debug.variableInspect.emptyTip` | ~~변수 인스펙터~~ — i18n 검증으로 교정 |
| Run History | 실행 기록 | `common.runHistory` | ~~실행 이력~~ — i18n 검증으로 교정 |
| Clear All (variables) | 모두 초기화 | `debug.variableInspect.clearAll` | ~~전체 초기화~~ |
| Reset to last run | 마지막 실행 값으로 재설정 | `debug.variableInspect.reset` | |
| Last Run | 마지막 실행 | `debug.lastRunTab` | |
| Run this step | 이 단계 실행 | `panel.runThisStep` | |
| Debug & Preview | 미리보기 | `common.debugAndPreview` | |

### 게시·웹앱 (i18n 검증, 2026-06-09)

> Publish 그룹 7p 작성 시 검증한 라벨.

| English (UI) | 한국어 | i18n 키 | 비고 |
|-------------|--------|---------|------|
| Batch Run (tab) | 일괄 실행 | `share.json` `generation.tabs.batch` | |
| Saved (tab) | 저장된 결과 | `share.json` `generation.tabs.saved` | |
| Start Chat (button) | 채팅 시작 | `share.json` `chat.startChat` | 대화 시작(Conversation Opener 기능명)과 구분 |
| New Chat | 새 채팅 | `share.json` `chat.newChat` | |
| Embed (entry) | 임베드 | `app-overview.json` `overview.appInfo.embedded.entry` | |
| Embed on Website | 웹사이트에 임베드하기 | `app-overview.json` `overview.appInfo.embedded.title` | |
| API Server | API 서버 | `app-api.json` `apiServer` | |
| API Secret Key | API 비밀 키 | `app-api.json` `apiKeyModal.apiSecretKey` | |

### 빌드·버전·앱 기능 (i18n 검증 완료, 2026-06-08)

> Build 그룹 7p 작성 시 검증한 라벨.

| English (UI) | 한국어 | i18n 키 | 비고 |
|-------------|--------|---------|------|
| Error Handling | 오류 처리 | `nodes.common.errorHandle.title` | |
| None (error strategy) | 없음 | `nodes.common.errorHandle.none.title` | |
| Default Value (error strategy) | 기본값 | `nodes.common.errorHandle.defaultValue.title` | |
| Fail Branch | 실패 분기 | `nodes.common.errorHandle.failBranch.title` | |
| Retry on Failure | 실패 시 재시도 | `nodes.common.retry.retryOnFailure` | |
| Current Draft | 현재 초안 | `common.currentDraft` | |
| Publish | 게시하기 | `common.publish` | |
| Publish Update | 업데이트 게시 | `common.publishUpdate` | |
| Version History | 버전 기록 | `versionHistory.title` | |
| Features (button) | 특징 | `common.features` | 앱 오른쪽 상단 버튼 |
| Conversation Opener | 대화 시작 | `feature.conversationOpener.title` | 채팅 플로우 전용 |
| Follow-up | 팔로우업 | `feature.suggestedQuestionsAfterAnswer.title` | 채팅 플로우 전용 |
| Text-to-Speech | 텍스트에서 음성으로 | `feature.textToSpeech.title` | 채팅 플로우 전용 |
| File Upload | 파일 업로드 | `feature.fileUpload.title` | |
| Citation | 인용 및 소유권 | `feature.citation.title` | 채팅 플로우 전용 |
| Content Moderation | 콘텐츠 모더레이션 | `feature.moderation.title` | 채팅 플로우 전용 |
| Parallel | 병렬 | `common.parallel` | |

### 워크스페이스 역할

| English (UI) | 한국어 | i18n 키 | 비고 |
|-------------|--------|---------|------|
| Owner | 소유자 | `members.owner` | |
| Admin | 관리자 | `members.admin` | 앱 빌드 및 팀 설정 관리 가능 |
| Editor | 편집자 | `members.editor` | 앱 빌드만 가능하고 팀 설정 관리 불가능 |
| Builder | 빌더 | `members.builder` | 자신의 앱을 구축 및 편집할 수 있습니다. i18n 발견 (2026-06-10) |
| Normal | 일반 | `members.normal` | ~~일반 멤버~~ → i18n 라벨은 "일반" |
| Knowledge Admin | 지식 관리자 | `members.datasetOperator` | 구 Dataset Operator |

### spx-agent 전용

| English | 한국어 | 비고 |
|---------|--------|------|
| Dashboard | 대시보드 | spx-agent 신규 챕터. **관리자용 워크스페이스 KPI** (부서별 리소스·모델 토큰·드릴다운). 앱 상세 탭의 "Monitoring(모니터링)"과 **다른 영역** — 모니터링은 앱 단위, 대시보드는 워크스페이스 단위. 한국어판에서 혼동 주의 |
| Department | 부서 | spx-agent 추가 |
| Visibility Scope | 가시성 범위 | private/department/custom/workspace |
| ACL (Access Control List) | ACL (접근 제어 목록) | 영문 + 괄호 한글 |
| Resource Ownership | 자원 소유권 | |
| Resource Permission | 자원 권한 | |
| Owner (resource) | 소유자 | Dify role Owner와 구분 |
| Principal | 주체 | ACL 대상 (사용자 또는 부서) |
| Audit Log | 감사 로그 | |
| RBAC | RBAC | 약어 유지 |
| SSO | SSO | 약어 유지 |
| Keycloak | Keycloak (단, **본문 노출 시 추상화**) | 본문에서는 "외부 시스템(예, Keycloak)" 또는 "관리 시스템"으로 표기 (전역 규칙 #6, 2026-06-04). 다른 IdP 사용 환경도 가정. 코드 식별자·NOTICE.md 같은 메타 정보에서만 고유명사 그대로 |
| 권한 설정 (챕터) | 권한 설정 | spx-agent 신규 챕터 (5/29 시점 "앱 권한 설정", 6/4 rename). 가시성·ACL·소유권의 **앱·지식·도구 공통 권한 모델 코어**. UI 진입은 각 도메인 상세 페이지(앱·지식·도구) "권한" 탭. 자세히는 [[references/spx-app-permissions-analysis]] |
| Drill-down | 드릴다운 | 대시보드 |
| KPI Card | KPI 카드 | 대시보드 |
| Collector | 수집기 | 감사로그 |
| dify-audit | dify-audit | 고유명사 (서비스명) |

### 권한 모델 — 가시성·액션·주체·효과 (A1 §11.2 기반, 2026-06-04 박제)

> [[references/spx-app-permissions-analysis]] §3·§4·§5에서 확정한 사용자 매뉴얼 어휘. 본 매핑은 권한 설정 챕터(A1)뿐 아니라 A2(지식 권한), A3(부서 ACL 부여), A4(워크스페이스 역할별 capability) 모두에 일관 적용.

#### 가시성 4단계 (Visibility Scope)

| 내부 값 | 한국어 라벨 | 묵시 부여 | 비고 |
|--------|-----------|--------|------|
| private | 비공개 | 없음 | 생성자 + 명시 ACL 대상자만 |
| department | 부서 공개 | 소유 부서 멤버 전원에 조회·실행 | 소유 부서 1개 지정 필수 |
| custom | 사용자 지정 | 없음 | 명시 ACL만 |
| workspace | 워크스페이스 공개 | 워크스페이스 전원에 조회·실행 | |

> 자동 부여는 `view`·`execute` 두 액션만. 나머지 6액션은 §명시 ACL 필요.

#### 액션 8종 (UI 체크박스 라벨 기준)

| 내부 값 | 한국어 라벨 | 적용 도메인 |
|---------|----------|-----------|
| view | 조회 | 앱·지식·도구 |
| edit | 편집 | 앱·지식·도구 |
| delete | 삭제 | 앱·지식·도구 |
| execute | 실행 | 앱·지식·도구 |
| publish | 발행 | **앱 전용** |
| duplicate | 복제 | **앱 전용** |
| manage_permission | 권한 관리 | 앱·지식·도구 |
| transfer | 소유권 이전 | 앱·지식·도구 |

#### 주체(Principal) 2종 + 효과(Effect) 2종

| 내부 값 | 한국어 라벨 | 비고 |
|---------|----------|------|
| user (principal) | 사용자 | ACL 부여 대상 |
| department (principal) | 부서 | ACL 부여 대상 (부서 멤버 일괄) |
| allow (effect) | 허용 | 기본값. UI는 allow만 제출 |
| deny (effect) | 거부 | 명시 차단. UI 미노출, API로만 등록 |

#### 권한 탭 UI 라벨

| 영역 | 한국어 라벨 |
|------|---------|
| 상단 카드 | 소유자 카드 |
| 좌측 메뉴 | 권한 탭(앱 상세) / 권한 메뉴 |
| ACL 부여 버튼 | 권한 부여 |
| 행 액션 — 수정 | 편집 |
| 행 액션 — 제거 | 취소 |

### 사용자·계정 — 로그인 옵션 (A4 §3.2 기반, 2026-06-04 박제)

> Personal Settings에서 노출되는 로그인 방식은 **systemFeatures 플래그**에 따라 4종 중 활성화. KC는 SSO 옵션을 통해 통합, 강제 X.

| 내부 값 | 한국어 라벨 | 비고 |
|---------|---------|------|
| email + password | 이메일 + 비밀번호 | |
| email + verification code | 이메일 + 코드 | |
| social login | 소셜 | |
| sso | SSO | 외부 시스템(예, Keycloak) 통합 — 전역 규칙 #6 적용 |

### 감사 로그 — 카테고리·행위자·액션 라벨 (B1 §10 기반, 2026-06-04 박제)

> [[references/spx-audit-log-analysis]] §10. 코드 `audit-meta.ts` 기준이며 **사용자 화면에 노출되는 한국어 라벨이 곧 매뉴얼 표기**.

#### 카테고리 / 행위자 유형

| 내부 값 | 한국어 라벨 |
|---------|----------|
| admin | 관리자 |
| user | 사용자 |
| security | 보안 |
| account (actor) | 관리자 |
| end_user (actor) | 앱 사용자 |
| api (actor) | API |
| system (actor) | 시스템 |

#### 액션 라벨 (전수)

| action | 라벨 | action | 라벨 |
|--------|------|--------|------|
| prompt_update | 프롬프트 수정 | app_create / app_update | 앱 생성 / 앱 수정 |
| api_token_create | API 토큰 발급 | workflow_publish / workflow_draft | 워크플로우 발행 / 초안 저장 |
| dataset_create | 데이터셋 생성 | member_join / member_change | 멤버 가입 / 멤버 변경 |
| document_upload | 문서 업로드 | workflow_node_execute | 워크플로우 노드 실행 |
| conversation_start | 대화 시작 | message_feedback | 답변 피드백 |
| message_send | 메시지 송수신 | provider_model_add / _update | 모델 추가 / 모델 변경 |
| workflow_execute | 워크플로우 실행 | document_delete / dataset_delete | 문서 삭제 / 데이터셋 삭제 |
| api_call | API 호출 | app_delete / api_token_delete | 앱 삭제 / API 토큰 삭제 |
| auth_failed | 인증 실패 | member_remove / member_role_change | 멤버 제거 / 멤버 역할 변경 |
| rate_limit_exceeded | Rate Limit 초과 | audit_login / _logout / _login_denied | audit 로그인 / 로그아웃 / 접근 거부 |
| | | audit_list_view / _detail_view / _export | audit 목록 조회 / 상세 조회 / 내보내기 |

> ⚠️ **"데이터셋" vs "지식" 표기 충돌 — 옵션 (가) 채택 (2026-06-04)**: `audit-meta.ts`의 `dataset_create` 라벨이 "데이터셋 생성"으로 박혀있어, 사용자가 사이드바 "지식" 메뉴에서 만든 자원이 감사 로그에선 "데이터셋"으로 표시됨. **글로서리는 "지식" 유지** + **매뉴얼 본문에서 "지식(데이터셋)" 혼용 안내**. 감사 로그 챕터(B1) 작성 시 "감사 로그에 '데이터셋'으로 표시되는 항목은 지식 베이스를 의미합니다" 같은 한 줄 안내 박는 것으로 처리. audit-meta.ts 코드 수정(옵션 나)은 보류. |

> 추가 용어 박을 때마다 본 표 갱신. 원본 영문 용어 대조는 `writing-guides/glossary.md` 참조.

## 커밋 메시지 (Dify 원본 컨벤션 따름)

```
{type}: {description}
```

- 소문자, 명령형 (`add`, `translate` — 과거형/3인칭 단수 금지)
- 마침표 없음, 72자 이내

| Type | 용도 | 예시 |
|------|------|------|
| `docs` | 새 내용 / 번역 추가·갱신 | `docs: add dashboard chapter (ko)` |
| `translate` | 번역 작업 | `translate: knowledge base intro (ko)` |
| `fix` | 오타, 깨진 링크, 잘못된 정보 | `fix: correct broken link in workflow page` |
| `feat` | 구조·도구 변경 | `feat: add ko language to docs.json` |
| `refactor` | 컨텐츠 변경 없는 재구조화 | `refactor: restructure use-dify section` |
| `chore` | 의존성, 설정 | `chore: bump mintlify 4.0.x` |
| `style` | 포맷팅 | `style: fix heading levels` |

## 빌드·개발 환경

- **빌드 도구**: Docusaurus (Mintlify에서 전환, 2026-06-02)
- **로컬 프리뷰**: `npm start` → `http://localhost:3000`
- **프로덕션 빌드**: `npm run build` → `build/` 정적 산출물
- **검색**: `@easyops-cn/docusaurus-search-local` (한국어, 오프라인)
- **MDXComponents 래퍼**: `src/theme/MDXComponents/` — Mintlify 전용 태그를 Docusaurus에서 렌더링
- **CSS**: `src/css/custom.css` — 테마 컬러 + 래퍼 컴포넌트 스타일

## i18n 구조

- **채택**: `ko/` 폴더 신설 + 수동 번역 (B안)
- `en/`은 수정하지 않음 (원본 보존 원칙)
- Docusaurus docs path는 `ko/`로 설정 — 빌드 시 `ko/` 내 MDX만 대상
- Phase 4 진척에 따라 자동화 파이프라인 검토

## 문서 관리 규칙

- 결정사항은 [[decisions]]에 기록
- 새 용어는 본 문서 글로서리에 즉시 추가
- 챕터별 진행은 [[progress]]에 표시
