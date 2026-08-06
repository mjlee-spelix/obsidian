# 번역 검증 종합 리포트 (ko/) — 정정판

> 생성 2026-06-15 · 대상 87개 파일 · 레이어2 서브에이전트 팬아웃 · 권위=도메인 i18n

## 📌 작업 컨텍스트 (다른 세션 단독 실행용 — 먼저 읽을 것)

이 리포트만으로 수정 작업을 할 수 있도록 경로·기준을 박아둡니다. (경로는 `/` 구분 절대경로 — Read 도구가 그대로 허용)

- **프로젝트 루트**: `C:/Users/Administrator/Projects/spx-agent-docs`
- **번역본(ko) 베이스**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/` + 각 항목 rel 경로
- **원본(en) 베이스**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/` + 동일 rel 경로 (paired 파일만)
- **i18n 권위 파일**: `C:/Users/Administrator/Projects/spx-agent/web/i18n/ko-KR/*.json`
- **편집 기준(필독)**: [[translation-verification-brief]] (4기준·심각도·전역 규칙·i18n 도메인 권위·문체) · [[conventions]] (글로서리·MDX·어미) · [[decisions]] (전역 규칙 #1~#5)
- **작업 절차**: ① 항목 ko 절대경로 Read → ② paired면 en 절대경로 Read해 대조 → ③ 수정 시 brief의 어미·MDX·용어 권위 준수 → ④ 용어 변경은 해당 도메인 i18n 재확인
- ⚠️ env var/배포 설정·이미지(Phase5)·플러그인 트리거·외부 Dify 리소스 누락은 **전역 규칙상 의도적 제거**이므로 본문 복원 금지(아래 🟢 오탐 참고)

## ⚠️ 정정 이력 (2026-06-15)

1차 브리프가 전역 규칙(decisions.md #1~#5)을 누락해 의도적 제거를 누락 위반으로 오판 → 브리프 정정 + ④누락 재분류 완료. ①②③ findings는 전역 규칙과 무관하여 전부 유효.

## 요약 (정정 후)

| 구분 | 수치 |
|---|---|
| 검증 파일 | 87 |
| 🔴 유효 findings | 152 (상 4 / 중 26 / 하 122) |
| 🟠 추출 갭 | 3 |
| 🟢 ④ 오탐 (의도적 제거, 수정 불필요) | 36 |
| 이상 없음 파일 | 8 |

## ✅ 처리 진행 현황 (2026-06-16 갱신)

> 본 리포트 기반 수정 작업의 완료 상태. 세부 결정은 [[decisions]] 결정 13·14, [[spx-edition-feature-gating]] 참조.

| 구간 | 상태 | 비고 |
|------|------|------|
| 🔴 상 4건 | **완료** | dashboard 색상·integrate 2건·schedule-trigger |
| 🟡 중 26건 | **완료** | workspace 중 6건 포함. 단 **quick-start #7은 무효 처리**(i18n='사용자 입력'으로 고쳤다가, 실제 캔버스 노드명='시작' 확인 후 원복 → [[decisions]] 결정 13) |
| 🟠 추출 갭 3건 | **완료** | `references/deployment-config-extracts.md`에 nodes/code·doc-extractor·template env 보완 |
| ⚪ 하 P1 17건 | **완료** | i18n 대조로 처리. 일부 오탐/보류(variable-assigner 쓰기모드·llm·key-concepts 축약) |
| ⚪ 하 P2 (용어·정보·링크·workspace) | **완료** | 객관적 항목 적용. 직역체(②)는 "명백한 것만" |
| 백로그 4건 | **완료** | 파라미터→매개변수(글로서리 전역) · DENY 비노출 · 매칭→일치/정제 유지 · model-providers(다중 자격 증명 보유=비용 최적화 복원, 로드 밸런싱 CE 미보유=제외). [[decisions]] 결정 14 |
| ⚪ 하 P3 (~32건) | **종결 (수정 불요)** | 2026-06-16 판정. frontmatter description은 fork 규칙상 **필수**(en에 없어도 ko는 유지)라 제거 시 오히려 위반 → 오탐. icon은 Docusaurus 미사용, 헤딩 H2없이H3는 원본 en 동일+원본 보존, `<Tip>`/콜아웃은 의도적 spx 추가물(유지). 빌드 무해 스타일 중 **이중 빈 줄만 선별 처리**(lesson-03 1건·lesson-04 2건). workspace/readme ASCII 트리 코드펜스는 이미 `text` 태그 적용됨. 외부 리소스 가능성 3건은 ko↔en 대조 검증 완료(2026-06-16) → 전부 #5 오탐: nodes/tools 플러그인 개발 가이드 링크 ko에 부재, lesson-08 Marketplace·Dify 브랜드 제거 후 재서술 완료, customer-service-bot community/SaaS 분기 제거(CE 단일 서술) 완료 |
| 빌드 | 전 구간 **통과** | `npm run build` broken link/MDX 오류 없음 |

**주요 결정 박제**: 노드명 Template="템플릿"(변환 폐기)·User Input="시작" / 다중 부서 멤버십 가능 확정 / spx-agent = CE self-hosted(유료·Enterprise 기능 제외) / 파라미터=매개변수 글로서리 적용.

## 🔴 상 — 반드시 수정 (유효) (4건)

### 1. [④누락/상] analytics-audit/dashboard/readme.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/analytics-audit/dashboard/readme.mdx`
- **en 원본**: `(원본 없음 — 신규 챕터)`
- **위치**: 66번째 줄 (모델별 토큰 사용량 섹션)
- **문제**: 모델별 토큰 차트의 색상 범주를 '기본 모델은 파란색, 로컬 모델은 청록색, 분류되지 않은 워크플로우는 회색' 세 가지로 설명하나, 실제 컴포넌트는 회색(미분류 워크플로우) 범주가 존재하지 않는 잘못된 사실 추가. 차트는 is_local boolean으로 기본 모델(파란색 #2E90FA)·로컬 모델(청록색 #15B79E) 두 범주만 구분함. 문서가 인용한 회색(#9CA3AF)은 데이터 막대 색이 아니라 축/값 라벨 텍스트 색일 뿐이며 '분류되지 않은 워크플로우'라는 카테고리는 소스에 없음.
- **권위확인**: 도메인 권위 = admin 컴포넌트 소스. web/app/components/admin/model-tokens-chart/index.tsx L11-12,L26: COLOR_DEFAULT(blue)/COLOR_LOCAL(teal) 두 색만, color = m.is_local ? COLOR_LOCAL : COLOR_DEFAULT. types.ts L5: is_local boolean만 존재. 회색(#9CA3AF)은 L58/L71 축·값 라벨 텍스트 색. 세 번째 '미분류/회색' 데이터 범주 없음. (비교: 바로 윗줄 62번째 줄의 부서별 오브젝트 색상 앱/지식/도구=파랑/청록/주황은 dept-objects-chart/index.tsx L15-17과 정확히 일치 → 색상 정보 자체는 작성자가 정확히 다뤘으므로 본 항목은 의도된 단순화가 아닌 사실 오류로 판단)
- **수정**: '분류되지 않은 워크플로우는 회색' 부분을 삭제하고 '기본 모델은 파란색, 로컬 모델은 청록색' 두 범주로 수정. 색 구분의 기준이 모델의 로컬 여부(is_local)임을 명시.

### 2. [①의미/상] knowledge/integrate-knowledge-within-application.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/integrate-knowledge-within-application.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/knowledge/integrate-knowledge-within-application.mdx`
- **위치**: L52 vs en L52
- **문제**: 원문 'matches the user's input text against the full text of the knowledge base'를 '지식의 전문(全文)과 매칭'으로 번역. '전문'은 중의적(full text vs 전문성)이며 full-text search 표준 한국어는 '전체 텍스트'. 의미는 '지식의 전체 텍스트'.
- **권위확인**: 글로서리: 전체 텍스트 검색(~~전문 검색~~ 금지). dataset.json hybrid 설명도 '전체 텍스트 검색' 사용(L159).
- **수정**: '지식의 전문과 매칭' → '지식의 전체 텍스트와 매칭'.

### 3. [④누락/상] knowledge/integrate-knowledge-within-application.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/integrate-knowledge-within-application.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/knowledge/integrate-knowledge-within-application.mdx`
- **위치**: L113~120 (Number 연산자 표) vs en L122~129
- **문제**: Number 표의 예시(Example) 정보가 통째로 누락됨. 원문은 '= → Example: =10 returns documents marked with exactly 10', '≠ → ≠5...', '> → >100...', '< → <50...', '≥ → ≥20...', '≤ → ≤200...', is empty/is not empty에도 예시 포함. 번역은 '정확한 숫자 일치','초과','미만' 등 설명만 남기고 예시 전부 제거. String/Date 표는 예시를 보존했으나 Number만 누락되어 비일관.
- **권위확인**: 원문 대조(en L122-129). self-host 유효 정보(연산자 사용 예시)이며 의도적 제거 맥락(Cloud/요금제) 아님.
- **수정**: Number 행에도 원문 예시 복원 (예: '= 정확한 숫자 일치. 예: `= 10` → 정확히 10으로 표시된 문서 반환' 등).

### 4. [④누락/상] nodes/trigger/schedule-trigger.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/nodes/trigger/schedule-trigger.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/nodes/trigger/schedule-trigger.mdx`
- **위치**: L73 (특수 문자 표 — `L` 행 예시 셀)
- **문제**: 원문 `L` 행 예시 셀은 `<br/><br/>`로 구분된 3개 진술(① `L`(일 필드)="Jan 31, April 30, or Feb 28 in a non-leap year", ② `L`(요일 필드)=Sunday, ③ `5L`(요일 필드)="the last Friday of the month")을 담고 있으나, 번역본 예시 셀은 ①과 ③만 있고 가운데 진술 **'`L` in the day-of-week field means Sunday'(요일 필드의 단독 `L`은 일요일)** 이 누락됨. cron 동작에 관한 self-host 유효 기술정보로, 단독 `L`을 요일 필드에 썼을 때 일요일로 해석된다는 구체 사실이 빠져 사용자가 설명 셀("단독으로 쓰면 마지막 요일")과의 관계를 오해할 수 있음.
- **권위확인**: 용어 아님(정보 누락). workflow.json nodes.triggerSchedule.* 키로 일정 트리거 라벨 일치 확인(nodeTitle="일정 트리거", title="일정"), 본 건은 표 셀 내용 대조 결과 누락.
- **수정**: 예시 셀의 `5L` 예시 앞에 누락된 진술을 복원: "**요일** 필드의 단독 `L`은 일요일을 의미합니다." (원문 `<br/><br/>` 구분 3진술 구조 유지)

## 🟡 중 — 수정 권장 (유효) (26건)

### 1. [③규칙/중] analytics-audit/audit-log/readme.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/analytics-audit/audit-log/readme.mdx`
- **en 원본**: `(원본 없음 — 신규 챕터)`
- **위치**: 본문 49행·83행·123행 (보안 카테고리/기록되는 활동 표/참고 — "호출 한도 초과")
- **문제**: 보안 이벤트 rate_limit_exceeded를 본문 전체에서 "호출 한도 초과"로 표기. 그러나 실제 화면에 노출되는 UI 라벨은 audit-meta.ts(권위) 기준 'Rate Limit 초과'임. 사용자가 화면에서 보는 라벨과 문서 용어가 불일치하여, 화면에서 'Rate Limit 초과' 배지를 찾을 때 혼동될 수 있음.
- **권위확인**: audit-meta.ts L36 ACTION_LABELS.rate_limit_exceeded = 'Rate Limit 초과' (권위 라벨). 브리프 글로서리 L75도 '라벨은 audit-meta.ts로 확인' 명시. 도메인 권위 = 'Rate Limit 초과'이며 문서의 '호출 한도 초과'와 다름.
- **수정**: 화면 라벨을 가리키는 맥락(특히 '기록되는 활동' 표 83행)에서는 'Rate Limit 초과(호출 한도 초과)' 식으로 실제 UI 라벨을 병기하거나, 최소 첫 등장 시 'Rate Limit 초과' 라벨을 명시. 설명 문맥(49·123행)의 '호출 한도 초과'는 자연어 설명으로 허용 가능하나 라벨 병기 권장.

### 2. [③규칙/중] build/goto-anything.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/build/goto-anything.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/build/goto-anything.mdx`
- **위치**: frontmatter L3
- **문제**: ko 파일에 description 필드가 있으나 원본 en frontmatter에는 title/icon만 있고 description이 없음. 브리프 규칙상 description은 원본 en에 있을 때만 허용되며, en에 없는데 ko에 추가하면 위반.
- **권위확인**: 원본 en goto-anything.mdx frontmatter 직접 확인: title, icon만 존재(L1-4), description 없음. i18n 무관(frontmatter 규칙).
- **수정**: description 필드를 제거하거나, 원본 정책상 ko 전체에 description 의무화 정책이 있다면 그 정책을 따름. 현 브리프 기준으로는 제거 권장.

### 3. [③규칙/중] build/mcp.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/build/mcp.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/build/mcp.mdx`
- **위치**: L96 문제 해결 — "서버 미설정"
- **문제**: 원문 troubleshooting의 실제 UI 오류 문자열 "Unconfigured Server"를 "서버 미설정"으로 번역. 제품 i18n의 동일 라벨은 "구성되지 않은 서버"라 인용된 UI 문자열이 실제 화면 표기와 불일치.
- **권위확인**: tools.json mcp.noConfigured = "구성되지 않은 서버". ko 본문은 "서버 미설정"으로 표기 → i18n 라벨 불일치 확인됨.
- **수정**: 인용 UI 라벨을 제품 표기와 맞춰 "구성되지 않은 서버"로 수정.

### 4. [③규칙/중] debug/step-run.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/debug/step-run.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/debug/step-run.mdx`
- **위치**: line 10 — "**실행**을 클릭하면"
- **문제**: 원문의 UI 버튼 "Run"을 "실행"으로 번역했으나, workflow.json의 실제 버튼 라벨은 "테스트 실행"(common.run)입니다. 굵게 표시된 부분이 명백히 클릭 대상 버튼을 가리키므로 실제 UI 라벨과 불일치합니다.
- **권위확인**: workflow.json common.run = "테스트 실행" (L207). 글로서리 UI 라벨·탭에도 "테스트 실행" 명시. i18n 확인됨.
- **수정**: "**실행**을 클릭하면" → "**테스트 실행**을 클릭하면"으로 수정 (실제 버튼 라벨과 일치).

### 5. [③규칙/중] getting-started/key-concepts.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/getting-started/key-concepts.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/getting-started/key-concepts.mdx`
- **위치**: 75행
- **문제**: '변수 할당기 노드'로 표기. 워크플로 노드 도메인 권위(workflow.json)에서 Variable Assigner는 '변수 할당자'이며, 글로서리도 ~~변수 할당기~~를 오표기로 명시함.
- **권위확인**: workflow.json blocks.assigner='변수 할당자', blocks.variable-assigner='변수 할당자' (nodes.variableAssigner.title은 '변수 할당'). i18n 확인 결과 '변수 할당기'는 미사용 라벨.
- **수정**: '변수 할당기 노드' → '변수 할당자 노드'로 수정.

### 6. [③규칙/중] getting-started/key-concepts.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/getting-started/key-concepts.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/getting-started/key-concepts.mdx`
- **위치**: 139행, 146행(표)
- **문제**: Normal 역할을 '일반 멤버'로 표기. i18n common.json의 권위 라벨은 'members.normal=일반'이며 글로서리도 ~~일반 멤버~~를 오표기로 명시함.
- **권위확인**: common.json members.normal='일반' (members.setMember='일반 멤버 설정'은 별개 액션 라벨). 멤버 역할 라벨 자체는 '일반'이 권위.
- **수정**: 역할명 라벨 '일반 멤버' → '일반'으로 통일(139행 나열, 146행 표 '일반 멤버(Normal)' → '일반(Normal)'). 설명문의 일반적 '멤버' 표현은 무방.

### 7. [①의미/중] getting-started/quick-start.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/getting-started/quick-start.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/getting-started/quick-start.mdx`
- **위치**: 전반 (L42·46 "시작 노드", L62 헤딩 "### 1. ... 시작 노드", 그리고 변수 프리픽스 시작/platform·시작/draft·시작/user_file·시작/language·시작/voice_and_tone 전체 — L133, L196, L215, L220, L312, L316, L427~430 등)
- **문제**: 원본은 이 노드를 신규 명칭인 'User Input' 노드로 부르고(헤딩 'User Input Node', 'Select the User Input node', 변수 참조 User Input/platform 등) 있는데, 번역본은 구(舊)명칭 '시작/시작 노드'로 되돌렸고 변수 프리픽스도 시작/xxx로 바꿨다. UI 라벨과 원문이 불일치하며, 독자가 화면에서 보는 노드명('사용자 입력')과 문서가 어긋난다.
- **권위확인**: workflow.json 확인: blocks.start='시작'(레거시), 그러나 이 버전에서 해당 노드는 'User Input'으로 리네임됨 — onboarding.userInputFull='사용자 입력 (원래 시작 노드)', nodes.start.outputVars.query='사용자 입력', panel.userInputField='사용자 입력 필드'. 즉 EN 'User Input'에 대응하는 i18n 라벨은 '사용자 입력'. 번역본의 '시작'은 EN과 i18n 현행 라벨 양쪽과 불일치.
- **수정**: 노드명과 변수 프리픽스를 '사용자 입력' / '사용자 입력/platform'(또는 원문대로 User Input/platform 유지)로 통일. 단, spx 빌드의 실제 UI 라벨이 '시작'으로 노출되도록 커스터마이즈됐다면 의도된 현지화일 수 있으므로 빌드 UI 확인 후 확정 권장.

### 8. [③규칙/중] knowledge/integrate-knowledge-within-application.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/integrate-knowledge-within-application.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/knowledge/integrate-knowledge-within-application.mdx`
- **위치**: 전반(L24,32,36,38,40,42,58,60,62,64 등 'Rerank' 전 등장)
- **문제**: 원문 'Rerank'를 전부 '리랭크'로 음차 번역. 지식 도메인 i18n 권위는 '재순위'(rerankSettings='재순위 설정', weightedScore.description='재순위 전략').
- **권위확인**: dataset.json L154 rerankSettings='재순위 설정', L182 '재순위 전략'; dataset-settings.json L37 '재순위 모델'. (dataset.json 자체도 L143 '리랭크', L159 '재랭크' 혼재하나 라벨 키 권위는 '재순위')
- **수정**: '리랭크' → '재순위'로 통일. 'Rerank 설정'='재순위 설정', 'Rerank 모델'='재순위 모델', 'Rerank 전략'='재순위 전략'.

### 9. [③규칙/중] knowledge/integrate-knowledge-within-application.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/integrate-knowledge-within-application.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/knowledge/integrate-knowledge-within-application.mdx`
- **위치**: L24, L26 (H3 헤더 '검색 방법')
- **문제**: 원문 'Retrieval Setting'(H3 헤더 및 'Context -- Retrieval Settings')을 '검색 방법'으로 번역. i18n 권위에서 'Retrieval Setting(s)'='검색 설정'이고 '검색 방법'은 별도 라벨(Retrieval Method)이라 두 개념이 혼동됨.
- **권위확인**: dataset.json L168 retrievalSettings='검색 설정', L156 retrieval.changeRetrievalMethod='검색 방법'; dataset-settings.json L38 form.retrievalSetting.title='검색 설정', L36 method='검색 방법'. 'Retrieval Setting'='검색 설정'이 권위.
- **수정**: H3 헤더 및 경로의 'Retrieval Setting(s)'은 '검색 설정'으로 수정. 'Retrieval Method'를 가리키는 부분만 '검색 방법' 유지.

### 10. [①의미/중] knowledge/integrate-knowledge-within-application.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/integrate-knowledge-within-application.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/knowledge/integrate-knowledge-within-application.mdx`
- **위치**: L153~177 FAQ vs en L177~199
- **문제**: 원문 FAQ 첫 항목이 '11.'로 시작(원문 오타)하나 번역은 '1.'로 정정함 — 개선이라 문제 아님. 원문 FAQ1 본문 'they can manually tweak'의 주어 혼동을 번역에서 '직접 조정할 수 있습니다'로 자연화한 것도 정상. 확인 차원의 하위 항목.
- **권위확인**: 원문 대조. 의미 변질 없음. 불확실성 명시 위해 등급 낮춤.
- **수정**: 번호 정정은 유지 가능. 별도 수정 불필요 — 참고용 기록.

### 11. [③규칙/중] knowledge/knowledge-pipeline/knowledge-pipeline-orchestration.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/knowledge-pipeline/knowledge-pipeline-orchestration.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/knowledge/knowledge-pipeline/knowledge-pipeline-orchestration.mdx`
- **위치**: ko L399 (<Note> 멀티모달 rerank 모델)
- **문제**: rerank 모델을 '멀티모달 리랭크 모델', '리랭킹'으로 번역. 그러나 이 Note는 지식 노드의 검색 설정(retrieval setting) 맥락(임베딩 모델이 멀티모달일 때 멀티모달 rerank 모델 선택)으로, 해당 도메인 i18n에서 'rerank'를 '재순위'로 라벨링한다. 같은 파일 내에서도 본문은 검색 설정/검색 방법 용어를 쓰는데 이 Note만 '리랭크'를 써 도메인 라벨과 불일치.
- **권위확인**: knowledge 도메인 dataset-settings.json `form.retrievalSetting.multiModalTip`='임베딩 모델이 멀티모달을 지원할 경우... 멀티모달 **재순위** 모델을 선택하세요' — en Note(L423-425)와 동일 맥락이며 권위 라벨은 '재순위'. (참고: workflow.json 노드='재정렬', common.json 모델제공자='재랭크'로 도메인별 상이하나, 본 Note는 지식 검색설정 도메인이므로 '재순위'가 권위.)
- **수정**: '멀티모달 리랭크 모델' → '멀티모달 재순위 모델', '리랭킹 및 검색 결과' → '재순위 및 검색 결과'로 교체하여 dataset-settings.json 라벨에 맞춤.

### 12. [③규칙/중] knowledge/manage-knowledge/maintain-dataset-via-api.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/manage-knowledge/maintain-dataset-via-api.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/knowledge/manage-knowledge/maintain-dataset-via-api.mdx`
- **위치**: 본문 8, 25행(제목), 29행
- **문제**: 'API 접근' / 'API Access'(영문)으로 표기되어 있으나, 지식 도메인 i18n 권위 라벨은 'API 액세스'입니다. 글로서리도 'API 액세스(~~API 접근~~)'로 명시. 25행 헤딩 '지식별 API 접근 관리', 8행 'API 접근이 기본적으로 활성화', 27행 '서비스 API를 통해 접근', 29행 UI 라벨 '**API Access**'(영문 미번역) 모두 해당.
- **권위확인**: common.json L94 'appMenus.apiAccess'='API 액세스', L95 apiAccessTip='이 지식 베이스는 서비스 API를 통해 액세스할 수 있습니다'. 글로서리 UI 라벨·탭 섹션도 'API 액세스(~~API 접근~~)' 확인.
- **수정**: 'API 접근' → 'API 액세스'로 통일. 29행 UI 버튼 라벨도 한국어 라벨 'API 액세스'(또는 'API 액세스(API Access)' 병기)로. 단 '접근'을 동사/일반 서술로 쓴 경우(예: '접근할 수 있습니다')는 i18n도 '액세스'를 쓰므로 동일하게 '액세스'로 맞추는 것이 일관적.

### 13. [④누락/중] knowledge/permissions/readme.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/knowledge/permissions/readme.mdx`
- **en 원본**: `(원본 없음 — 신규 챕터)`
- **위치**: 라인 22 vs 라인 68-70 (FAQ '부서에 새 멤버를 추가했는데...')
- **문제**: 내적 일관성 충돌. 본문 라인 22는 '권한 탭의 변경은 자동으로 검색 권한에도 반영됩니다. 사용자가 별도로 동기화 작업을 수행할 필요는 없습니다'라고 단정하지만, FAQ 라인 70은 '부서 멤버 변경이 지식 검색 권한에 자동으로 반영되지 않는 경우가 있습니다 ... 강제 재동기화를 요청'이라고 반대로 안내한다. 독자가 자동 반영 여부를 상반되게 이해할 수 있다(특히 부서 멤버 변경 경로).
- **권위확인**: i18n 무관(동작 서술 일관성 문제). 글로서리/i18n로 판정 불가한 본문 내적 모순.
- **수정**: 본문 라인 22를 '권한 탭에서의 직접 변경은 자동 반영되며, 부서 멤버 추가/이동 등 간접 변경은 반영이 지연되거나 재동기화가 필요할 수 있습니다'처럼 예외를 명시하도록 보강해 FAQ와 정합을 맞추시기 바랍니다.

### 14. [④누락/중] nodes/http-request.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/nodes/http-request.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/nodes/http-request.mdx`
- **위치**: L56-62 (인증 섹션)
- **문제**: 원문은 각 인증 방식에 실제 설정 코드값을 병기함: No Auth `(type: "no-auth")`, API Key `(type: "api-key")`, Basic `(type: "basic")`, Bearer `(type: "bearer")`, Custom `(type: "custom")`. 번역본은 이 type 코드값을 모두 제거함. 이는 SaaS/요금제 맥락이 아니라 self-host에서도 유효한 실제 구성값(설정 식별자)이므로 정상 제거 대상이 아니며 기술 정보 누락에 해당.
- **권위확인**: workflow.json L482-491: nodes.http.authorization.no-auth='없음', basic='기본', bearer='Bearer', custom='사용자 정의', api-key='API 키' — UI 라벨 번역은 일치. 다만 type 코드값(no-auth/api-key 등)은 i18n 라벨이 아닌 원문의 기술 주석으로, 누락 여부는 ④ 정보 누락 기준으로 판단(라벨 위반 아님).
- **수정**: 각 인증 유형에 원문의 `type: "..."` 코드값을 괄호로 복원하거나, 최소한 self-host 사용자가 설정 시 참조할 식별자를 유지할 것.

### 15. [③규칙/중] nodes/list-operator.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/nodes/list-operator.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/nodes/list-operator.mdx`
- **위치**: L44 (파일 속성 탭) "전송 방식(Transfer Method)"
- **문제**: Transfer Method를 "전송 방식"으로 번역했으나, 노드 도메인 권위 파일 workflow.json의 transfer_method 라벨은 "전송 방법"으로 통일되어 있음. UI 라벨과 불일치.
- **권위확인**: workflow.json — nodes.agent.outputVars.files.transfer_method = "전송 방법...", nodes.tool.outputVars.files.transfer_method = "전송 방법...", common.humanInputEmailTip = "...이메일(전달 방법)...". listFilter 전용 transfer_method 키는 없으나 노드 전반에서 transfer_method="전송 방법"으로 일관됨. "전송 방식"은 미사용.
- **수정**: "전송 방식(Transfer Method)" → "전송 방법(Transfer Method)"으로 수정.

### 16. [③규칙/중] nodes/variable-assigner.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/nodes/variable-assigner.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/nodes/variable-assigner.mdx`
- **위치**: 본문 line 31 — '**값 설정** — 상위 워크플로우 노드에서 소스 데이터를 선택합니다.'
- **문제**: 원문 UI 라벨 '**Set Variable**'을 '값 설정'으로 번역했으나, 노드 도메인 권위 i18n(workflow.json)의 실제 라벨은 '변수 설정'이다. 같은 노드 내 '변수' 라벨(line 29)과의 구분을 위해 정확한 UI 라벨을 써야 일관성·검색성이 유지된다.
- **권위확인**: workflow.json 키 'nodes.assigner.setVariable' = '변수 설정' 확인. ko 본문은 '값 설정'으로 불일치.
- **수정**: '**값 설정**'을 '**변수 설정**'으로 교체 (i18n 라벨과 일치). 영문 병기가 필요하면 '변수 설정(Set Variable)'.

### 17. [③규칙/중] publish/developing-with-apis.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/publish/developing-with-apis.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/publish/developing-with-apis.mdx`
- **위치**: L41 텍스트 생성 앱 단락
- **문제**: 원문 "the developer's settings in the Dify Prompt Arrangement page"를 "오케스트레이트 페이지의 설정"으로 번역. 그러나 publish 도메인 i18n(app-api.json completionMode.info)은 이 동일 개념(텍스트 생성 모델 파라미터·프롬프트 템플릿을 설정하는 페이지)을 'SPX Agent Prompt Engineering' / 'Prompt Eng'로 표기한다. 글로서리는 '오케스트레이트'를 UI 라벨로 캐시했으나, 이 파일 도메인의 실제 i18n 라벨은 'Prompt Engineering'이므로 라벨 불일치.
- **권위확인**: app-api.json completionMode.info(L44)='SPX Agent Prompt Engineering 에서 설정한 모델 매개변수와 프롬프트 템플릿', chatMode.info(L30)='SPX Agent Prompt Eng 의 설정'. 'orchestrat/오케스트레이' 검색은 app-api.json·share.json에서 미발견. common.json appMenus.apiAccess='API 액세스'(본문 사용 라벨은 정확). 도메인 i18n 권위 = 'Prompt Engineering'.
- **수정**: "오케스트레이트 페이지의 설정"을 i18n 라벨에 맞춰 "Prompt Engineering 페이지의 설정"(또는 '프롬프트 엔지니어링 페이지')으로 교체. 영문 병기 시 첫 등장에 한국어(English) 형식 적용 고려.

### 18. [③규칙/중] tutorials/article-reader.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/tutorials/article-reader.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/tutorials/article-reader.mdx`
- **위치**: L13, L19 (배울 내용 / 사전 준비)
- **문제**: 'Chatflow'가 영어 그대로 노출됨. 첫 등장 시 한국어 표준 라벨로 병기되어야 함. 본문 전체에서 'Chatflow'로만 표기.
- **권위확인**: app.json typeSelector.advanced/types.advanced='채팅 플로우' 확인. 글로서리도 Chatflow=채팅 플로우(챗플로우X). i18n 권위와 일치.
- **수정**: 첫 등장(L19 또는 L13)을 '채팅 플로우(Chatflow)'로 병기 후 이후 '채팅 플로우'로 통일. 최소한 한국어 라벨 사용 권장.

### 19. [③규칙/중] tutorials/build-ai-image-generation-app.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/tutorials/build-ai-image-generation-app.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/tutorials/build-ai-image-generation-app.mdx`
- **위치**: ko L170 (질문 2 섹션) — "**기능 추가(Add Feature) - 콘텐츠 검열(Content Moderation)**"
- **문제**: Content Moderation을 "콘텐츠 검열"로 번역. UI 라벨 권위와 불일치.
- **권위확인**: app-debug.json feature.moderation.title="콘텐츠 모더레이션", operation.addFeature="기능 추가". 즉 i18n 권위는 "콘텐츠 모더레이션"(검열 아님). "기능 추가"는 일치.
- **수정**: "콘텐츠 검열" → "콘텐츠 모더레이션(Content Moderation)"으로 수정해 i18n UI 라벨과 일치시키기. 본문 다른 곳의 "콘텐츠 검열"도 동일 처리.

### 20. [③규칙/중] tutorials/customer-service-bot.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/tutorials/customer-service-bot.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/tutorials/customer-service-bot.mdx`
- **위치**: ko 75 '전문 검색', 그리고 line 93~95 헤딩 '#### 리콜 테스트', 95
- **문제**: (1) 'full-text retrieval'을 '전문 검색'으로 번역. i18n dataset.json은 '전체 텍스트 검색'. (2) 'Recall Test'를 '리콜 테스트'로 음차. i18n은 '검색 테스트'.
- **권위확인**: (1) dataset.json L158 'retrieval.full_text_search.title'="전체 텍스트 검색", L92 indexingMethod.full_text_search="전체 텍스트". 글로서리도 '전체 텍스트 검색(~~전문 검색~~)'. → '전문 검색' 위반 확정. (2) common.json L156 datasetMenus.hitTesting="검색 테스트", dataset-hit-testing.json L25 title="검색 테스트". 'Recall Test' 대응 UI 라벨은 '검색 테스트'. → '리콜 테스트' 위반 확정
- **수정**: '전문 검색' → '전체 텍스트 검색', '리콜 테스트'(헤딩·본문 둘 다) → '검색 테스트'로 수정

### 21. [③규칙/중] workspace/app-management.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/workspace/app-management.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/workspace/app-management.mdx`
- **위치**: 84행, 116행
- **문제**: "API 접근"으로 번역. 원문 "API access". 글로서리 UI 라벨 규칙은 'API 액세스(~~API 접근~~)'로 '접근'을 명시적 금지 표현으로 지정. 두 곳 모두 '접근' 사용.
- **권위확인**: common.json appMenus.apiAccess="API 액세스", appMenus.apiAccessTip="...API를 통해 액세스..."; app.json deleteAppConfirmContent/tracing 키 모두 "액세스" 사용. i18n 권위는 '액세스'로 일관 확인됨.
- **수정**: "API 접근"을 "API 액세스"로 수정 (84행 '게시된 웹앱과 API 액세스', 116행 '...API 액세스가 즉시 사라집니다' 형태).

### 22. [③규칙/중] workspace/departments/readme.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/workspace/departments/readme.mdx`
- **en 원본**: `(원본 없음 — 신규 챕터)`
- **위치**: L10 "일반 멤버에게는 메뉴 자체가 보이지 않습니다"
- **문제**: 역할 라벨을 "일반 멤버"로 표기. 워크스페이스 역할 권위 라벨은 "일반"(Normal)이며 글로서리도 ~~일반 멤버~~를 명시적 비권장으로 둠. 문맥상 멤버 일반을 가리키는 일반 명사로도 읽힐 여지는 있으나, 바로 앞 문장이 "관리자(Admin) 이상"이라는 역할 등급을 말하므로 역할 라벨로 해석됨.
- **권위확인**: common.json members.normal="일반", members.normalTip="앱 사용만 가능...". 글로서리 워크스페이스 역할 항목과 일치(일반, ~~일반 멤버~~).
- **수정**: 역할을 지칭하는 경우 "일반(Normal) 역할" 또는 "일반 역할 사용자"로 정정. 단순 '일반적인 멤버'를 뜻한 것이면 "일반 사용자" 등으로 표현해 역할 라벨과 구분.

### 23. [③규칙/중] workspace/departments/readme.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/workspace/departments/readme.mdx`
- **en 원본**: `(원본 없음 — 신규 챕터)`
- **위치**: L47-48 "멤버 관리 모달의 **제거** 버튼은..."
- **문제**: 멤버 관리 모달의 버튼 라벨을 "제거"로 표기했으나 실제 UI 라벨은 "제외"임. 사용자가 화면에서 찾을 버튼명과 불일치.
- **권위확인**: common.json rbac.department.membersModal.remove="제외". (참고로 members.deleteMember="멤버 삭제", members.removeFromTeam="팀에서 제거"는 다른 화면) 도메인 권위는 rbac.department.* = "제외".
- **수정**: "**제외** 버튼"으로 정정.

### 24. [③규칙/중] workspace/departments/readme.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/workspace/departments/readme.mdx`
- **en 원본**: `(원본 없음 — 신규 챕터)`
- **위치**: L36-37 "부서의 **비활성** 버튼을 클릭하면"
- **문제**: 버튼 라벨을 "비활성"으로 표기. 실제 동작/버튼 라벨은 "비활성화"이며 "비활성"은 상태값 라벨(inactive)임. L35 헤딩 "부서 비활성화"는 맞으나 본문 버튼명만 "비활성"으로 어긋남.
- **권위확인**: common.json rbac.department.deactivate="비활성화"(동작), rbac.department.inactive="비활성"(상태). 버튼은 deactivate="비활성화".
- **수정**: "부서의 **비활성화** 버튼"으로 정정.

### 25. [③규칙/중] workspace/model-providers.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/workspace/model-providers.mdx`
- **en 원본**: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/workspace/model-providers.mdx`
- **위치**: 53~131행("커스텀 모델" 다수), 102행 Tab title "커스텀 모델", 103·105·116·123·129행 등
- **문제**: 원본 "Custom Model"을 ko에서 "커스텀 모델"로 번역했으나, 워크스페이스 도메인 i18n(common.json)의 권위 라벨은 "사용자 지정 모델"임. 같은 문서 내 다른 Tab("사전 정의 모델")은 i18n과 일치시키면서 Custom만 "커스텀"으로 음차해 라벨 불일치.
- **권위확인**: common.json modelProvider.auth.customModelCredentials="사용자 지정 모델 자격 증명", customModelCredentialsDeleteTip 등에서 "사용자 지정 모델" 확인. 글로서리 가시성 섹션 custom="사용자 지정"과도 일치.
- **수정**: "커스텀 모델"을 "사용자 지정 모델(Custom Model)"로 통일. 첫 등장 시 영문 병기.

### 26. [③규칙/중] workspace/permissions/readme.mdx
- **ko 파일**: `C:/Users/Administrator/Projects/spx-agent-docs/ko/use-spx-agent/workspace/permissions/readme.mdx`
- **en 원본**: `(원본 없음 — 신규 챕터)`
- **위치**: 117행: "...**편집자(Editor)** 와 **일반 멤버(Normal)** 의 접근을 제어하는 용도입니다."
- **문제**: 역할 라벨 'Normal'을 '일반 멤버'로 병기. 워크스페이스 역할 도메인 권위(common.json members.normal)는 라벨 자체가 '일반'이며, '일반 멤버'는 액션 문구(setMember='일반 멤버 설정')에 한정됨. 글로서리도 Normal=일반(~~일반 멤버~~ X)으로 캐시.
- **권위확인**: common.json members.normal="일반" (L244). members.setMember="일반 멤버 설정"(L259)은 액션 문구로 별개. 역할 라벨 권위는 '일반' 확정.
- **수정**: '일반 멤버(Normal)' → '일반(Normal)'로 수정. (편집자/Editor 병기는 members.editor=편집자와 일치하여 정상)

## 🟠 추출 갭 — deployment-config-extracts.md에 추가 (3건)

본문 제거는 규칙상 정상. env var/배포 설정이 추출 파일에 누락 → **본문이 아닌 추출 파일**에 보완.

- **nodes/code.mdx** (en: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/nodes/code.mdx`) @ ko 끝부분 (en 118-126행 'Self-Hosted Setup' 섹션 누락)
  - '## Self-Hosted Setup'(예: '## 셀프 호스팅 설정') 섹션을 복원하여 docker-compose 명령 코드블록(bash)과 Docker 요구·격리 설명을 번역 추가할 것.
- **nodes/doc-extractor.mdx** (en: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/nodes/doc-extractor.mdx`) @ ko L91 (외부 의존성) vs en L94
  - '`UNSTRUCTURED_API_URL`과 `UNSTRUCTURED_API_KEY` 환경변수로 Unstructured API 서비스가 설정되어 있어야 합니다' 식으로 환경변수명을 복원
- **nodes/template.mdx** (en: `C:/Users/Administrator/Projects/spx-agent-docs/en/use-dify/nodes/template.mdx`) @ 106행 (## 출력 제한 / 원본 107행)
  - "템플릿 출력은 최대 **80,000자**로 제한됩니다(환경변수 `TEMPLATE_TRANSFORM_MAX_LENGTH`로 조정 가능). 대규모 템플릿 출력에서 메모리 문제를 방지하고..." 형태로 환경변수 안내를 복원.

## ⚪ 하 — 확인 (유효, 파일별 122건)

| 파일 | 건수 | 기준 | 요지 |
|---|---|---|---|
| build/shortcut-key.mdx | 3 | ③규칙 | 원본 en frontmatter에 description이 없는데 / 'Hand tool (pan)'→'손 도구 (이동)', 'Sel / 원본 en frontmatter의 icon: "keyboard" |
| nodes/http-request.mdx | 3 | ④누락,②자연스러움 | 원문 'SSL certificate verification is / 원문 L48 'Connect timeout: ... (defau / 번역본은 'Response Body'를 '응답 내용(Body)' |
| nodes/llm.mdx | 3 | ④누락,①의미,③규칙 | 원문 'GPT-4 and Claude 3.5 handle com / 원문 'High detail / Low detail'(상세 수준 / 이미지 마크다운 '![](/images/llm-memory.pn |
| publish/webapp/web-app-settings.mdx | 3 | ④누락,①의미 | 원문 description은 "branding, basic ac / 원문 "This text appears on the app's  / 원문 Step 제목 "Review access controls" |
| tutorials/article-reader.mdx | 3 | ①의미,④누락,③규칙 | 원문 'enabling file upload in the fun / 원본 'File upload is supported in v0. / 'knowledge base'를 '지식 베이스'로 표기. 글로서 |
| tutorials/customer-service-bot.mdx | 3 | ③규칙,①의미 | 검색 설정(지식 도메인)에서 'reranking model'을  / en 'The community edition of Dify u / 'Direct Reply Node'를 '직접 답변(Direct |
| tutorials/workflow-101/lesson-04.mdx | 3 | ③규칙,④누락 | 이중(연속) 빈 줄 존재. 179행 본문과 183행 코드블록 사 / 원본은 코드블록을 <CodeGroup>…</CodeGroup>( / '역색인(Inverted Index)'으로 병기. dataset |
| workspace/departments/readme.mdx | 3 | ③규칙,④누락 | 워크스페이스 설정 사이드바 메뉴명을 "멤버", "부서"로 적었으 / 버튼/모달 명칭을 "멤버 관리"로 표기했으나 i18n 버튼 라벨 / 문서는 한 사용자가 "여러 부서"에 속할 수 있음을 전제로 서술 |
| workspace/model-providers.mdx | 3 | ④누락,③규칙 | 원본의 <Info>Default Config refers to  / 원본 자격 증명 다중 등록 시나리오 3종(Environment  / 받침 없는 명사("추가") 뒤 목적격 조사가 "를"이어야 하는데 |
| workspace/personal-settings/readme.mdx | 3 | ③규칙 | 본문·표에서 프로필 항목 라벨을 '표시명'으로 표기. 실제 Di / '인터페이스 언어'라는 표현 사용. common.json의 권위 / 원본 en(personal-account-management.m |
| analytics-audit/audit-log/readme.mdx | 2 | ③규칙 | 워크스페이스 역할 'Normal'을 '일반 멤버'로 지칭. 브리 / knowledge base를 '지식 베이스'(띄어쓰기)로 표기. |
| build/goto-anything.mdx | 2 | ③규칙 | title 값이 따옴표로 감싸짐("Go to Anything") / H2 없이 H3(###)로 시작. 다만 이는 원본 en이 동일하 |
| build/mcp.mdx | 2 | ③규칙 | 원본 en frontmatter에는 title·icon만 있고  / 상대 링크 대상 ko/use-spx-agent/publish/p |
| build/orchestrate-node.mdx | 2 | ③규칙 | 원본 en frontmatter에는 title과 icon만 있고 / 문서가 H2(## )로 시작하며 H1/문서 도입 본문 없이 바로 |
| build/version-control.mdx | 2 | ③규칙,④누락 | 원본 en 파일의 frontmatter에는 title과 icon / 원본 en frontmatter의 'icon: "layer-gr |
| knowledge/create-knowledge/introduction.mdx | 2 | ④누락 | 원본 step 1은 데이터 소스를 3종 제시(로컬 파일 업로드  / 원본에 없는 <Tip> 콜아웃(권한 설정 안내)이 번역본에 추가 |
| knowledge/integrate-knowledge-within-application.mdx | 2 | ②자연스러움,③규칙 | 'cross-language/cross-lingual'을 '크로 / 원본 en frontmatter에 icon('puzzle-pie |
| knowledge/knowledge-pipeline/create-knowledge-pipeline.mdx | 2 | ③규칙 | 원문 'cleaning strategies'를 '정제 전략'으로 / 본문 prose에서 'matching'을 '매칭'으로 표기('직 |
| knowledge/knowledge-pipeline/readme.mdx | 2 | ④누락 | 원문 'to optimize data processing for / 원문 'built-in pipeline templates tha |
| knowledge/manage-knowledge/introduction.mdx | 2 | ④누락,③규칙 | 원본 en에는 없는 <Tip> 콜아웃(권한 설정 챕터로의 교차  / 원본 en은 [Select the Index Method](.. |
| knowledge/manage-knowledge/maintain-knowledge-documents.mdx | 2 | ③규칙 | 'Top K'를 라인 49는 '상위 K(Top K)', 라인 9 / 'match'를 동사형으로 '매칭되면'(49)·'잘 매칭되지'( |
| knowledge/permissions/readme.mdx | 2 | ③규칙,②자연스러움 | '운영자' 역할 라벨이 워크스페이스 역할 i18n에 존재하지 않 / '~를 유지하기 위해' 목적절 + 무생물 주어('권한 탭의 변경 |
| knowledge/test-retrieval.mdx | 2 | ③규칙 | 원본은 [Configure the Retrieval Settin / 링크 텍스트를 '인덱싱 방법 설정'으로 표기. 대상 페이지 제목 |
| monitor/annotation-reply.mdx | 2 | ④누락,③규칙 | 원문 en에 없는 콜아웃·설명이 다수 추가됨. (1) <Info / 동일 개념 'annotation reply'를 본문 전반에서는 |
| monitor/logs.mdx | 2 | ④누락,③규칙 | 원본은 단순 4개 불릿(Conversation Timeline  / description 자체는 원본 en 3행에 존재하므로 규칙 |
| nodes/human-input.mdx | 2 | ①의미,③규칙 | 원문은 'the workflow automatically end / 원본 en frontmatter에는 'icon: user-mag |
| nodes/ifelse.mdx | 2 | ③규칙 | 원본 en title은 "If-Else"이나 ko title은  / "크다/작다"(Greater than/Less than), "같 |
| nodes/question-classifier.mdx | 2 | ③규칙 | 본문에서 분류 단위를 일관되게 '카테고리'로 옮겼다. 원문 pr / 원본 en frontmatter의 icon: "sitemap" |
| nodes/tools.mdx | 2 | ④누락,③규칙 | 원본 마지막 문단의 안내 링크 'For detailed guid / 원본 frontmatter의 icon: "wrench" 키가 k |
| nodes/variable-assigner.mdx | 2 | ③규칙,①의미 | 원문 'Operation Mode(s)'를 '쓰기 모드'로 번역 / 원문 'Add all elements from another a |
| publish/readme.mdx | 2 | ④누락,③규칙 | 원본의 4번째 불릿 '**Rate limits** - Prote / 원본 frontmatter의 `icon: "rocket"` 키가 |
| publish/webapp/embedding-in-websites.mdx | 2 | ④누락 | 원문 line 47 `baseUrl: 'https://udify / 원문 line 64 `name: "John Doe", // Va |
| tutorials/build-ai-image-generation-app.mdx | 2 | ④누락,②자연스러움 | 원문 "Of course, this format is just  / 원문 "Prompts are the soul of the Age |
| tutorials/workflow-101/lesson-01.mdx | 2 | ①의미 | 원문 'We are going to take you from Z / 원문 1번은 'Go [Dify](https://dify.ai/) |
| tutorials/workflow-101/lesson-02.mdx | 2 | ①의미,③규칙 | 원문은 예시 변수명을 `Destination`, `Travel  / 본문·frontmatter에서 핵심 개념을 '워크플로우'로 표기 |
| tutorials/workflow-101/lesson-03.mdx | 2 | ③규칙 | 코드블록 앞에 빈 줄이 2줄 연속(이중 빈 줄)으로 들어가 '이 / 본문에서 'Model Provider'를 '모델 공급자'로 번역 |
| tutorials/workflow-101/lesson-05.mdx | 2 | ①의미,③규칙 | 원문은 입력란에 구체적으로 'Dify'를 입력하고 IF 로직을  / IF/ELSE 비교 연산자 라벨을 일치(Is)/불일치(Is No |
| tutorials/workflow-101/lesson-08.mdx | 2 | ④누락,①의미 | 원본 라인 107 "Select a model that supp / 원본 "In the Marketplace, find Dify A |
| tutorials/workflow-101/lesson-10.mdx | 2 | ③규칙 | '탐색에서 열기'(Open in Explore) 및 '탐색에서  / frontmatter에 description 없음(en에도 없으 |
| workspace/readme.mdx | 2 | ③규칙 | 대시보드·감사 로그 상세 링크가 절대경로 마크다운 링크(`/us / ASCII 트리 다이어그램 펜스 코드블록에 언어 태그 없음. 브 |
| analytics-audit/dashboard/readme.mdx | 1 | ③규칙 | 약어 'KPI'가 첫 등장 시 한국어 병기 없이 사용됨(브리프 |
| build/additional-features.mdx | 1 | ③규칙 | 원본 en frontmatter에는 title/icon만 있고 |
| build/predefined-error-handling-logic.mdx | 1 | ③규칙 | 원본 en frontmatter에는 title과 icon만 있고 |
| debug/error-type.mdx | 1 | ③규칙 | 원본 frontmatter의 icon: "circle-xmark |
| debug/step-run.mdx | 1 | ③규칙 | 원본 en frontmatter는 title/icon 구성이고 |
| getting-started/introduction.mdx | 1 | ③규칙 | 원본 en frontmatter에는 title/mode/icon |
| getting-started/key-concepts.mdx | 1 | ④누락 | 원문(en L119)의 'persist over multi-tu |
| getting-started/quick-start.mdx | 1 | ③규칙 | 원본 title '30-Minute Quick Start'의 ' |
| knowledge/create-knowledge/setting-indexing-methods.mdx | 1 | ②자연스러움 | 검색 모델의 명칭은 '재순위 모델(Rerank Model)'로 |
| knowledge/knowledge-pipeline/knowledge-pipeline-orchestration.mdx | 1 | ③규칙 | 문서가 H2 없이 H3(### 인터페이스 상태)로 시작한 뒤 L |
| knowledge/knowledge-pipeline/manage-knowledge-base.mdx | 1 | ③규칙 | 본문이 H2 없이 H3(###)로 시작함. 다만 원본 en도 동 |
| knowledge/metadata.mdx | 1 | ①의미 | 원본 'This guide aims to help you und |
| knowledge/readme.mdx | 1 | ③규칙 | '[지식 파이프라인으로 생성](./knowledge-pipeli |
| nodes/agent.mdx | 1 | ③규칙 | tool parameter 일반 용어를 '파라미터'로 옮겼는데, |
| nodes/code.mdx | 1 | ④누락 | en 116행 'Check the available packag |
| nodes/doc-extractor.mdx | 1 | ④누락 | 원문 마지막 문장 'The extracted text maint |
| nodes/iteration.mdx | 1 | ③규칙 | 원본 en frontmatter의 `icon: "arrows-r |
| nodes/knowledge-retrieval.mdx | 1 | ③규칙 | 원본 en frontmatter에는 title과 icon만 있고 |
| nodes/list-operator.mdx | 1 | ①의미 | 원문 "creating seamless multi-modal u |
| nodes/loop.mdx | 1 | ③규칙 | "변수 할당자"가 본문 첫 등장이지만 영문 병기(Variable |
| nodes/template.mdx | 1 | ③규칙 | 노드명을 "템플릿 변환"으로 표기. 노드 도메인 권위인 work |
| nodes/trigger/webhook-trigger.mdx | 1 | ③규칙 | 번역본은 'Parameters'를 '파라미터'로 옮겼으나 노드 |
| nodes/user-input.mdx | 1 | ④누락 | en 원문 "guide end users on the expec |
| publish/publish-mcp.mdx | 1 | ③규칙 | 원본 frontmatter에는 icon: "network-wir |
| publish/webapp/chatflow-webapp.mdx | 1 | ③규칙 | 'Citations'를 '출처 인용'으로 번역. 해당 publi |
| publish/webapp/workflow-webapp.mdx | 1 | ③규칙 | 동일 용어 WebApp 표기가 제목 '웹앱'과 본문 '웹 앱'으 |
| tutorials/simple-chatbot.mdx | 1 | ②자연스러움 | '단순히 빠르게 만드는 것을 넘어 ... 채택하는 것입니다' 구 |
| tutorials/workflow-101/lesson-06.mdx | 1 | ③규칙 | UI 섹션 라벨을 i18n workflow.json 권위 라벨과 |
| tutorials/workflow-101/lesson-07.mdx | 1 | ①의미 | 원본 title은 "Enhance Workflows (Plugi |
| tutorials/workflow-101/lesson-09.mdx | 1 | ②자연스러움 | 외부 Jinja 문서 링크의 표시 텍스트 'Template De |
| workspace/app-management.mdx | 1 | ④누락 | 원본 en에 description이 있어 ko에 descript |
| workspace/permissions/readme.mdx | 1 | ④누락 | FAQ 답변에서 '거부(DENY)' 행 개념을 처음 언급하나, |

## ⚪ 하 — 우선순위 분류 (2026-06-15 추가)

> 위 하 표(요지 잘림) 기준으로 P1/P2/P3 분류. 정확한 라인·문구는 실제 수정 시 원문/i18n으로 최종 확인. 상·중 152건은 처리 완료(단 finding #7 quick-start는 실제 UI 확인 결과 노드명이 '시작'이라 무효 처리·원복).

### 우선순위 기준

2개 축으로 판정:

| 축 | 높음 | 낮음 |
|----|------|------|
| 영향(독자) | 화면 라벨 불일치로 못 찾음 / 의미 오해 / 실제 동작·제약 정보 누락 | 빌드·메타·내부 품질, 독자 인지 영향 거의 없음 |
| 확정성·비용 | i18n·원문으로 명확 + 기계적 치환 | 주관적 문체 판단 / "원본 en도 동일" |

기준(criterion)별 기본 티어:

| 유형 | 티어 |
|------|------|
| ① 의미 변질(오역), ③ UI 라벨·연산자·노드명 i18n 불일치, ④ 실제 기술정보(예시·제약·동작) 누락 | **P1** |
| ② 직역체, ③ 표기 통일(병기·띄어쓰기·링크 텍스트), ④ 부가 설명·코드 예시값 보완 | **P2** |
| ③ frontmatter icon/description, 헤딩 레벨(원본 동일), ④ 원본에 없는 spx 의도적 추가(`<Tip>`·콜아웃), 빌드 무해 스타일 | **P3** |

- P1=먼저, P2=일괄 정리, P3=보류/선택. 한 파일에 여러 티어 섞이면 열 때 함께 처리.
- 권장 순서: **workspace 중 6건 → P1 → P2 → P3(선택)**

### ⚠️ workspace 중 등급 6건 (하보다 우선 — 앞서 작업서 제외했던 i18n 라벨 확정 건)

| 파일 | finding |
|------|---------|
| workspace/app-management.mdx | API 접근→API 액세스 (중 #21) |
| workspace/departments/readme.mdx | 일반 멤버→일반(#22), 제거→**제외** 버튼(#23), 비활성→**비활성화** 버튼(#24) |
| workspace/model-providers.mdx | 커스텀 모델→**사용자 지정 모델** (#25) |
| workspace/permissions/readme.mdx | 일반 멤버→일반 (#26) |

### 🔴 P1 — 먼저 (사용자 가시·확정, ~17건)

| 파일 | 핵심 사유 |
|------|----------|
| nodes/ifelse.mdx | "크다/작다" 비교 연산자 i18n 라벨 대조 |
| nodes/variable-assigner.mdx | 'Operation Mode'→'쓰기 모드' 라벨 |
| nodes/template.mdx | 노드명 "템플릿 변환" (workflow.json 권위 대조) |
| nodes/human-input.mdx | 'workflow automatically ends' 의미 |
| nodes/llm.mdx | High/Low detail 비전 설정 정보 누락 |
| getting-started/key-concepts.mdx | 'persist over multi-turn' 메모리 정보 |
| tutorials/customer-service-bot.mdx | 'Direct Reply'→직접 답변 라벨, reranking 도메인 라벨 |
| tutorials/workflow-101/lesson-05.mdx | IF/ELSE Is/Is Not 연산자 라벨 |
| tutorials/workflow-101/lesson-06.mdx | UI 섹션 라벨 i18n |
| publish/webapp/chatflow-webapp.mdx | 'Citations'→'출처 인용' 라벨 |
| analytics-audit/audit-log/readme.mdx | 역할 'Normal'→'일반 멤버' (처리한 중1과 별개 위치) |
| knowledge/permissions/readme.mdx | 존재하지 않는 '운영자' 역할 라벨 |
| workspace/departments/readme.mdx | 사이드바 메뉴명·버튼/모달 명칭 i18n 라벨 2건 + "여러 부서 소속" 사실 확인 1건 |
| workspace/personal-settings/readme.mdx | 프로필 라벨 '표시명', '인터페이스 언어' common.json 대조 2건 |

### 🟡 P2 — 일괄 정리 (품질·일관성, ~56건)

- **용어·표기 통일(③)**: knowledge/manage-knowledge/maintain-knowledge-documents(Top K 병기·match), knowledge/knowledge-pipeline/create-knowledge-pipeline(정제 전략·매칭), nodes/agent(파라미터), nodes/trigger/webhook-trigger(Parameters), nodes/question-classifier(카테고리), nodes/loop(변수 할당자 병기), publish/webapp/workflow-webapp(웹앱/웹 앱), tutorials/workflow-101/lesson-03(모델 공급자), lesson-10(탐색에서 열기), build/shortcut-key(Hand tool), tutorials/article-reader(지식 베이스 띄어쓰기), analytics-audit/dashboard(KPI 병기)
- **직역체·자연스러움(②)**: tutorials/simple-chatbot, knowledge/integrate(크로스 랭귀지), knowledge/create-knowledge/setting-indexing-methods, knowledge/metadata, nodes/list-operator, tutorials/build-ai-image-generation-app, tutorials/workflow-101/lesson-01·02·07·09
- **정보 보완(④)**: nodes/http-request(SSL·timeout·Response Body), nodes/code, nodes/doc-extractor, nodes/user-input, knowledge/knowledge-pipeline/readme, knowledge/create-knowledge/introduction(데이터 소스 3종), publish/readme(Rate limits — SaaS quota 여부 확인), publish/webapp/web-app-settings, publish/webapp/embedding-in-websites(코드 예시값), tutorials/workflow-101/lesson-08
- **링크 텍스트·대상(③)**: knowledge/readme, knowledge/test-retrieval, knowledge/manage-knowledge/introduction, build/mcp(상대 링크 대상 — broken이면 P1 승격)
- **workspace 추가**: workspace/model-providers(Default Config·자격 증명 3종·조사 오류), workspace/readme(상세 링크 절대→상대경로), workspace/permissions(FAQ DENY 내적 일관성), workspace/personal-settings(en 구조 대조 1건)

### ⚪ P3 — 보류/선택 (~32건)

- **frontmatter icon/description**(Docusaurus 미사용, 원본 동일): build/additional-features·predefined-error-handling·orchestrate-node·version-control·goto-anything, debug/error-type·step-run, getting-started/introduction·quick-start(title 숫자), nodes/iteration·knowledge-retrieval, publish/publish-mcp, mixed 파일의 icon finding, workspace/app-management(description 완전성)
- **헤딩 레벨 H2 없이 H3**(원본 en 동일): knowledge/knowledge-pipeline/orchestration·manage-knowledge-base, goto-anything, orchestrate-node
- **원본에 없는 spx 의도적 추가**(유지 검토): knowledge/create-knowledge/introduction·manage-knowledge/introduction의 `<Tip>`(권한 챕터 교차 안내), monitor/annotation-reply 콜아웃
- **빌드 무해 스타일**: 이중 빈 줄·코드펜스 태그(lesson-03·04 등), workspace/readme ASCII 트리 코드블록 언어 태그
- **외부 리소스 가능성**(전역 규칙 #5 재확인 후 오탐이면 제외): nodes/tools 'For detailed guide' 링크, lesson-08 Marketplace, customer-service-bot 'community edition'

### 갱신 요약

| 버킷 | 건수(대략) |
|------|-----------|
| workspace 중 (선처리) | 6 |
| 🔴 P1 (먼저) | ~17 |
| 🟡 P2 (일괄) | ~56 |
| ⚪ P3 (보류/선택) | ~32 |

## 🟢 ④ 오탐 — 전역 규칙상 의도적 제거 (36건, 수정 불필요)

| 분류 | 건수 | 파일 |
|---|---|---|
| 오탐:이미지(Phase5) | 18 | publish/publish-mcp.mdx, tutorials/customer-service-bot.mdx, knowledge/integrate-knowledge-within-application.mdx, knowledge/metadata.mdx, publish/webapp/chatflow-webapp.mdx, publish/webapp/workflow-webapp.mdx, workspace/model-providers.mdx, build/additional-features.mdx … |
| 오탐:env추출완료 | 6 | knowledge/create-knowledge/import-text-data/readme.mdx, knowledge/knowledge-pipeline/knowledge-pipeline-orchestration.mdx, knowledge/manage-knowledge/maintain-knowledge-documents.mdx, monitor/logs.mdx, nodes/knowledge-retrieval.mdx |
| 오탐:외부리소스(#5) | 5 | tutorials/workflow-101/lesson-10.mdx, knowledge/readme.mdx, build/shortcut-key.mdx, knowledge/manage-knowledge/maintain-dataset-via-api.mdx, tutorials/workflow-101/lesson-07.mdx |
| 오탐:플러그인삭제 | 4 | nodes/trigger/overview.mdx, nodes/trigger/webhook-trigger.mdx |
| 오탐:self-host분기 | 3 | knowledge/knowledge-pipeline/knowledge-pipeline-orchestration.mdx, knowledge/create-knowledge/setting-indexing-methods.mdx, knowledge/create-knowledge/chunking-and-cleaning-text.mdx |

## 이상 없음 (8)

`debug/history-and-logs.mdx` · `debug/variable-inspect.mdx` · `knowledge/knowledge-pipeline/publish-knowledge-pipeline.mdx` · `knowledge/knowledge-pipeline/upload-files.mdx` · `nodes/answer.mdx` · `nodes/output.mdx` · `nodes/parameter-extractor.mdx` · `nodes/variable-aggregator.mdx`
