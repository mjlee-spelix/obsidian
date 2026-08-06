# 챕터 작성 체크리스트

> Phase 4 본문 작성 절차. **3단계**: 그룹 사전 점검(1회) → 페이지 작성(매 페이지) → 그룹 마감(1회).
> 작성: 2026-06-05 / 재구성: 2026-06-08 (그룹·페이지·마감 분리, N/A 반복 제거)

---

## 1. 그룹 사전 점검 (그룹당 1회)

> 같은 그룹(예: Nodes 23p, Build 6p) 작업 시작 전에 **한 번만** 수행. 결과를 메모해두면 이후 페이지마다 재확인 불필요.

### 1.1 매트릭스 일괄 확인
- [ ] [[scope-mapping]] 해당 그룹 전체 행 확인 — 액션(유지·번역/부분 수정/삭제/변환)·우선순위·비고
- [ ] **삭제 대상 분리** — [[scope-mapping#확정 삭제]] 표에서 본 그룹 해당 페이지 목록 확정. 작업 대상에서 제외
- [ ] **부분 수정 페이지 식별** — [[scope-mapping#spx-agent 분석으로 확인된 부분 수정 보강 사항]] 표에서 본 그룹 해당 확인. 있으면 §A 추가 적용 대상으로 표시

### 1.2 decisions 영향 일괄 확인
- [ ] [[decisions]] 결정 1~10 중 본 그룹에 영향 있는 결정 식별
  - 결정 1 — Marketplace + 플러그인 제거
  - 결정 2 — 외부 연결 제거
  - 결정 3 — MCP 유지
  - 결정 4 — inbound 3건 유지
  - 결정 5 — KC 추상화 (모든 페이지 영향)
  - 결정 6 — team-members-management 변환
  - 결정 7 — CI/CD 표시
  - 결정 8 — 문서 전체 사용자 대상
  - 결정 9 — "앱 권한 설정" → "권한 설정" rename
  - 결정 10 — 사이드바 Option α
- [ ] 영향 있는 결정 메모 (예: "Nodes — 결정 1로 Plugin Trigger 삭제, 나머지 N/A")

### 1.3 i18n 그룹 라벨 일괄 추출
- [ ] [[conventions#챕터 도메인 → i18n 파일 매핑]]에서 본 그룹의 우선 검색 파일 확인
- [ ] 해당 i18n 파일에서 **그룹 전체에 쓰일 라벨을 한 번에 추출** — UI 라벨(버튼·탭·컬럼명) + 도메인 개념어(핵심 동사·상태값·수치 단위)
  - 예: Knowledge → `dataset-hit-testing.json` 전체 + `dataset.json` retrieval/indexing 키 + `dataset-settings.json` index/retrieval 키
- [ ] 추출 결과를 **그룹 라벨 테이블**로 정리 (영문 키 → 한국어 값 → 비고)
- [ ] [[conventions#용어집]] 대조 — 충돌 시 i18n 승, 글로서리 즉시 갱신
- [ ] 미등재 용어 발견 시 글로서리에 즉시 추가

### 1.4 폴더·사이드바 확정
- [ ] CLAUDE.md §사이드바 구조 (Option α) 확인
- [ ] 본 그룹 폴더 경로 패턴 확정 (예: `ko/use-spx-agent/nodes/<slug>.mdx`)
- [ ] `sidebars.js` 카테고리 확인 또는 신설

### 1.5 분석본 필요 여부 판단
- [ ] [[#4. 그룹별 빠른 시작 가이드]]에서 본 그룹 항목 확인 — 분석본 인용 필요 여부, 특이사항
- [ ] 필요 시 해당 분석본(A1~C1) 읽기. 불필요하면 "분석본 N/A" 메모

---

## 2. 페이지 작성 (페이지마다)

> 매 페이지마다 수행. §1에서 확인한 그룹 공통 정보는 재확인 불필요.

### 2.1 원본 읽기 + 전역 규칙 스캔
- [ ] 원본 `en/` 파일 읽기
- [ ] 전역 규칙 #1~#6 해당 콘텐츠 있는지 스캔 ([[scope-mapping#전역 적용 규칙]])
  - #1 SaaS 플랜 → #2 Cloud 분기 → #3 Sandbox/Production → #4 env var → #5 Dify 브랜드 → #6 KC → #7 액션 무관 적용 → **#8 문장 적합성**
  - 있으면 번역 시 제거/추출/추상화 적용
- [ ] **문장 적합성 판단 (전역 규칙 #8)** — 원본 문장마다 "spx-agent 사용자가 이 안내대로 실행할 수 있는가?" 확인. 제품에 없는 기능을 권고하는 문장, 자체 호스팅 환경과 무관한 운영 맥락은 번역 대상에서 제외. 키워드 없이 SaaS 맥락이 스며든 문장(예: "로그 익명화를 검토하시기 바랍니다")에 특히 주의

### 2.2 i18n 라벨 대조 (§1.3 테이블 활용)
- [ ] §1.3에서 뽑아둔 **그룹 라벨 테이블**에서 본 페이지에 등장할 용어를 대조
- [ ] 테이블에 없는 새 라벨이 필요하면 그때만 i18n 파일 추가 grep → 테이블에 추가
- [ ] 미등재 용어 발견 시 글로서리에 즉시 추가

### 2.3 본문 작성
> [[conventions]] 전체 참조. 아래는 핵심만 요약.

- [ ] **frontmatter** — `title` 필수, `description` 원본에 있으면 동일하게, `sidebar_label` 필요 시
- [ ] **합쇼체** — `~할 수 있습니다`, `~됩니다`. 청유·명령은 `~하시기 바랍니다`
- [ ] **영문 병기** — 전문 용어 첫 등장 시 `한국어(English)`, 이후 한국어만
- [ ] **MDX 포맷** — 헤딩 순서(H2→H3→H4), Bold(UI 요소·핵심 용어), `<Frame>` + caption + alt, 간격(빈 줄 1개), 내부 링크 상대경로
- [ ] **피해야 할 패턴** — 과도한 불릿, UI 반복 설명, 기능 중심 도입(`~해줍니다`), 군더더기
- [ ] **콜아웃** — `<Info>`/`<Tip>`/`<Note>`/`<Warning>` 용도에 맞게, 남용 금지
- [ ] 부분 수정 페이지면 → **§A 추가 적용**
- [ ] 신규 챕터면 → **§B 추가 적용**
- [ ] 데이터 조회·시각화 챕터면 → **§C 추가 적용**

### 2.4 직역체 검수 (3패스)
> [[conventions#직역체 회피 (translationese)]] + [[translationese-guide]] 참조. **이 두 문서를 열어놓고** 대조할 것.

- [ ] **1패스 — 의미 번역**: 문단 단위로 의미 파악 후 원문을 보지 않고 한국어로 작성 (문장 1:1 매핑 금지)
- [ ] **2패스 — 마커 검출**: grep + **소리 내 읽기**(M2 무생물 주어·M3 명사 체인·M10 직역 어휘는 grep으로 못 잡음)
  ```bash
  grep -nE "을 위한|를 위한|로부터|를 통해|에 의해|되어집니다|상호 배타|할 수 있게 해|제공합니다|의 .+의 " ko/use-spx-agent/<경로>/<파일>.mdx
  ```
- [ ] **3패스 — 원문 대조**: 섹션별로 원문과 맞춰 누락 확인 (표·콜아웃·이미지 통째 빠짐 방지)

### 2.5 페이지 검증
- [ ] **i18n 사후 검증** — 본문에 사용한 라벨을 §1.3 그룹 라벨 테이블과 재대조 (§2.2에서 놓친 충돌 검출)
- [ ] **전역 규칙 잔존 grep**
  ```bash
  grep -nEi "Free|Pro plan|Enterprise|Subscription|Billing|Dify|dify\.ai|langgenius|Powered by|Keycloak|KC SSO" ko/use-spx-agent/<경로>/<파일>.mdx
  ```
- [ ] **빌드 검증** — `npm run build` (매 페이지 또는 3~5페이지 단위. 본 페이지 발 broken link 0건 확인)
- [ ] **sidebars.js 등록**
- [ ] **deferred 발생 시 즉시 메모** (dangling link, 소급 교정 필요 등)

---

## 3. 그룹 마감 (그룹 완료 시 1회)

> 그룹 내 모든 페이지 작성 완료 후 수행.

### 3.1 빌드 최종 검증
- [ ] `npm run build` — 그룹 전체 반영 상태에서 최종 통과 확인
- [ ] `<Card href>` 절대경로 확인 (Card.jsx는 broken link 검사 우회)

### 3.2 메타 갱신 (⚠️ 상태와 로그는 **두 문서**에 따로 반영)
- [ ] **[[progress]] 상태 갱신** (현황만) — ① 챕터별 상태 표 그룹 행(⏳→✅) ② 상단 "📍 현재 상태 한눈에" 블록(날짜·완료 그룹 수·다음 할 일)
- [ ] **[[progress-archive]] 작업 로그 기록** (시간순 이력) — `### YYYY-MM-DD (그룹명 작성)` 헤딩 + 한 작업 요약. ※ 로그는 progress가 아니라 **archive**에 (2026-06-09 분리 이후)
- [ ] [[conventions]] 신규 용어 일괄 확인 — 작업 중 추가한 용어가 빠짐없이 등재되었는지
- [ ] [[decisions]] 작업 중 새 결정 발생했으면 박제

### 3.3 deferred 일괄 정리
- [ ] 작업 중 메모한 deferred 항목 → [[progress]] deferred 섹션에 박제
- [ ] dangling link 목록 정리 (어떤 페이지 작성 시 해소되는지 명시)

---

## A. 부분 수정 페이지 추가 적용

> §2.3에서 해당 시에만 적용. 액션이 "부분 수정"인 페이지 전용.

- [ ] [[scope-mapping#spx-agent 분석으로 확인된 부분 수정 보강 사항]] 표의 본 페이지 보강 내용 적용
- [ ] 분석본 §X 인용 (해당 분석본의 인용 매핑 표 참조)
- [ ] 신규 챕터 키워드 티저·링크 (Introduction이라면 4개 신규 챕터)

## B. 신규 챕터 추가 적용

> §2.3에서 해당 시에만 적용. spx-agent 전용 신규 챕터 전용.

- [ ] 분석본의 인터리브 임시 작성 부분 참조하여 챕터 구조 확장
- [ ] 분석본의 한국어 매핑 글로서리에 즉시 반영
- [ ] 코드 검증된 사실만 본문에 기술
- [ ] 톤 일관성 — 같은 사이드바 그룹의 기존(번역) 챕터를 기준점으로 어미·콜아웃·헤딩 맞춤

## C. 데이터 조회·시각화 챕터 서술

> §2.3에서 해당 시에만 적용. 대시보드·감사로그·모니터링(Analysis·Logs) 전용.
> [[conventions#데이터 조회·시각화 챕터 서술 원칙]] 참조.

- [ ] 차트·카드·컬럼 나열로 끝내지 않음 — 각 요소마다 ①의미 ②왜 중요 ③업무 활용 3단 서술
- [ ] 화면에 보이는 라벨·수치 반복 X → "읽는 법·활용" 제공
- [ ] (대시보드) 증감 배지·부서별/모델별 비교 인사이트
- [ ] (감사로그) 각 로그 항목 설명 + 조회/필터/내보내기 시나리오

---

## 4. 그룹별 빠른 시작 가이드

> §1.5에서 참조. 그룹별 특이사항·분석본 필요 여부·i18n 파일 요약.

### Nodes 23p
- 매트릭스: 유지·번역 P2 (Plugin Trigger 삭제)
- 분석본: 불필요
- i18n: `workflow.json`
- decisions 영향: 결정 1 (Plugin Trigger 삭제)만. 나머지 N/A

### Build 6p
- 매트릭스: 유지·번역 P2 (MCP 포함)
- 분석본: 불필요
- i18n: `workflow.json`
- decisions 영향: 결정 3 (MCP 유지). 나머지 N/A

### Debug 4p
- 매트릭스: 유지·번역 P2
- 분석본: 불필요
- i18n: `workflow.json` `debug.*` 키 — **변수 검사** (인스펙터 X)
- decisions 영향: N/A
- ✅ 완료 (2026-06-05)

### Publish 6p
- 매트릭스: 유지·번역 + 부분 수정 혼재. Web App Access·Publish to Marketplace 삭제
- 분석본: Publish Overview만 — 결정 1 적용 (Marketplace 제거) + 권한 제약 한 줄
- i18n: `share.json`, `app.json`, `app-api.json`
- decisions 영향: 결정 1 (Marketplace 전체 제거), 결정 4 (embedding·APIs 유지)

### Monitor 3p
- 매트릭스: Analysis 부분 수정 (제목 "모니터링"), Logs·Annotation Reply 유지·번역. integrations 7p 삭제
- 분석본: 불필요 (B2 폐기)
- i18n: `app-debug.json`, `app-log.json`, `app-annotation.json`
- decisions 영향: 결정 2 (integrations 삭제)
- Analysis·Logs → **§C 데이터 시각화 서술 적용**

### Knowledge ~14p
- 매트릭스: 유지·번역 + 부분 수정 3p. 외부 연결 5건 + Rate Limit 삭제
- 분석본: A2 (Knowledge 권한) + A1 (공통 권한 모델)
- i18n: `dataset.json`, `dataset-creation.json`, `dataset-documents.json`, `dataset-hit-testing.json`, `dataset-settings.json`, `pipeline.json`, `dataset-pipeline.json`
- decisions 영향: 결정 2 (외부 연결 삭제)
- **"데이터셋" vs "지식" 표기 충돌** — 감사 로그 챕터 안내 한 줄 (옵션 가)

### Workspace 4p
- 매트릭스: 유지·번역 2p + 부분 수정 2p. Plugins·Team Members·Subscription·API Extension 삭제
- 분석본: A4 (Workspace 재검토)
- i18n: `common.json`, `layout.json`, `oauth.json`, `register.json`
- decisions 영향: 결정 1·2·5·6 다수 해당

### Get Started 3p
- 매트릭스: 부분 수정 P1 (3p 전부)
- 분석본: C1 + A1·A3·대시보드·B1 (신규 챕터 키워드 티저)
- i18n: `common.json`, `login.json`, `layout.json`
- decisions 영향: 결정 5 (KC 추상화), 결정 8 (전체 사용자 대상)
- 인터리브 3p 이미 작성됨 → 정밀화·deferred 검토

### Tutorials 14p
- 매트릭스: 유지·번역 P2. Twitter Chatflow 삭제
- 분석본: 불필요
- i18n: (튜토리얼 전용 키 없음, 범용 `workflow.json` + `app.json`)
- decisions 영향: 결정 2 (Twitter 삭제)

### 신규 챕터
- 워크스페이스 그룹: 사용자·부서 관리 / 권한 설정 → §B 적용
- 통계·감사 그룹: 대시보드 / 감사로그 → §B + §C 적용
- 인터리브 1p 모두 작성됨 → 확장 + deferred 처리

---

## 5. 참조 문서 빠른 링크

| 문서 | 용도 |
|------|------|
| [[scope-mapping]] | 매트릭스·전역 규칙·확정 삭제·보강 사항 |
| [[conventions]] | MDX 포맷·문체·UI 라벨 검증·글로서리·직역체 회피 절차 |
| [[translationese-guide]] | 직역체 Before/After 사례집 (2패스 검수 시 **열어놓고** 대조) |
| [[decisions]] | 결정 1~10 사유·대안·후속 트리거 |
| [[progress]] | 챕터별 상태·진행 |
| CLAUDE.md §사이드바 구조 | Option α 폴더 경로 |
| references/spx-*.md | 챕터별 분석 정전 |
