# 검수 규칙 브리프 (단일 출처)

> 검수 에이전트가 **이 파일을 읽고 그대로 적용**한다. 규칙 수정은 **여기 한 곳만** 고친다.
> ⚠️ **모든 finding은 아래 규칙 중 하나를 근거로 cite해야 한다.** 근거를 못 대면 "잘못됐다"고 주장하지 말고 area 4(확인 필요)로 사람에게 넘긴다. ("애매한지"를 느낌으로 판단하지 말고, "근거 규칙을 댈 수 있는지"로 판단)
> 규칙의 정확한 문구가 필요하면 아래 **권위 문서를 직접 열어** 확인한 뒤 cite한다. 이 브리프는 권위 문서의 빠른 참조이지 대체물이 아니다.

## 0. 권위 문서 (애매하면 직접 읽어라 — 이게 정답)

| 문서 | 절대경로 | 용도 |
|------|---------|------|
| 체크리스트 | `C:\Users\Administrator\Projects\spx-agent-docs\.claude\docs\chapter-writing-checklist.md` | §2 페이지 검수 절차(원문읽기→전역규칙→i18n→본문→직역체3패스→사후검증), §A/B/C |
| conventions | `C:\Users\Administrator\Projects\spx-agent-docs\.claude\docs\conventions.md` | 문체·존댓말·영문병기·i18n검증·피해야할패턴·직역체·데이터시각화·글로서리 |
| decisions | `C:\Users\Administrator\Projects\spx-agent-docs\.claude\docs\decisions.md` | 결정 1~12 (삭제/유지/추상화/보강 근거) |
| scope-mapping | `C:\Users\Administrator\Projects\spx-agent-docs\.claude\docs\scope-mapping.md` | 전역규칙 #1~#8 전문, 확정 삭제 23건, 페이지별 액션, 부분수정 보강 |
| translationese | `C:\Users\Administrator\Projects\spx-agent-docs\.claude\docs\translationese-guide.md` | 직역체 Before/After 사례 |

**원칙: 추측 금지. finding을 내려면 근거 규칙(아래 A~E 중 하나)을 cite하라. "삭제/유지/보강" 판단은 decisions·scope-mapping을 직접 확인해 cite. 근거 못 대면 area 4.**

---

## 1. 검수 절차 (체크리스트 §2를 페이지마다 적용)

1. **원문(en) 읽기 + 전역 규칙 #1~#8 스캔** (아래 A)
2. **i18n 라벨 대조** — 본문 볼드 UI 라벨을 i18n(권위)과 대조 (아래 E)
3. **본문 규칙 점검** — 문체·존댓말·영문병기·MDX·콜아웃·피해야할패턴 (아래 E)
4. **직역체 3패스** — ①의미 번역인가 ②마커 grep + 소리내 읽기 ③원문 대조(빠진 표·콜아웃·이미지)
5. **사후 검증** — 전역 규칙 잔존 grep, 표기 일관성
6. 데이터 조회·시각화 챕터면 **§C 서술 원칙** 추가 (아래 E)

---

## 2. 이미 결정된 처리 — findings로 올리지 마라 (가장 중요)

> ⚠️ 아래 A~D에 해당하는 "처리"는 규칙대로 된 것이라 **정상**이다. 절대 "누락/문제"로 보고하지 마라.

### A. 전역 규칙 #1~#8 (scope-mapping). 원문에 있던 게 ko에 없어도 아래면 의도된 처리
1. **SaaS 플랜 제거**: Free/Pro/Team/Enterprise plan, Subscription, Billing, Seat, Quota
2. **"Cloud version에서는…" 분기** → CE 단일 서술
3. **Sandbox/Production 워크스페이스(SaaS)** → 단일 환경. (단 `langgenius/dify-sandbox` 보안 컨테이너는 보존)
4. **"자체 호스팅 한정" 분기 제거** (spx는 항상 self-hosted). env var/배포 설정 본문은 `references/deployment-config-extracts.md`로 **추출**(완전 삭제 아님). "관리자" 표현 회피
5. **Dify 브랜드 전면 제거**: 브랜드명·로고·외부채널(github/forum/discord/twitter)·dify.ai/docs.dify.ai 링크·"Powered by Dify". (예외: NOTICE.md, 내부 컨테이너명 dify-audit 등 보존)
6. **외부 인증(Keycloak) 추상화**: "외부 시스템(예, Keycloak)". 로그인은 "SSO로 로그인하는 경우" 분기 만들지 말 것 → 본문에 Keycloak/KC SSO 직접 노출은 area 3
7. 액션이 "유지·번역"이어도 위 항목 있으면 제거/추출
8. **문장 적합성**: "spx-agent 사용자가 이 안내대로 실행 가능한가?" 없는 기능 권고·SaaS 운영맥락 문장 삭제. ⚠️ **부분 제거** — 요금/pricing만 덜고 유효 정보(토큰 소비·기본값·동작 제약)는 보존·재서술

### B. 확정 삭제 23건 (의도된 삭제 — ko에 없는 게 정상)
- **Marketplace + 플러그인(결정1)**: Plugin Trigger, publish-to-marketplace, workspace/plugins, "마켓플레이스에서 설치/추가" 안내, web-app-access(Enterprise)
- **외부 연결 outbound(결정2)**: Monitor integrations 7종(LangSmith/Langfuse/Opik/Weave/Arize/Phoenix/Aliyun), 지식 외부 import(Sync from Notion/Website, Authorize Data Source), 외부 KB(Connect External KB, External Knowledge API), Twitter Chatflow, workspace/api-extension 3종
- **기타**: knowledge-request-rate-limit(Cloud quota), subscription-management, cloudflare-worker

### C. 유지 확정 (헷갈리지 말 것 — 있어야 정상)
- **inbound 3건 유지·번역(결정4)**: publish/webapp/embedding-in-websites, publish/developing-with-apis, knowledge/.../maintain-dataset-via-api ← 본문·"## API Reference" 유지 대상
- **MCP 유지(결정3)**: build/mcp, publish/publish-mcp
- **모델 제공자 설치 유지**(사용자 환경마다 백엔드 다름)

### D. 의도된 보강 (원문에 없어도 정상 — "원문에 없는 추가"로 문제삼지 말 것)
- 부분수정 페이지의 **"권한 설정 섹션 + 신규 챕터 링크"** 추가(2026-06-02 격상): getting-started 3p, publish/README, workspace/readme, knowledge(create-knowledge/introduction, create-knowledge-pipeline, manage-knowledge/introduction)
- monitor/analysis 제목 "Analysis" → "모니터링"

---

## 3. conventions 글쓰기 규칙 (E)

### 문체·존댓말 (확정)
- **합쇼체**: `~합니다` / `~할 수 있습니다`(선택적) / `~해야 합니다`(필수)
- **청유·명령(행동 안내) 표준 어미 = `~하시기 바랍니다`** (2026-06-05 확정)
  - ⚠️ **`~하시기 바랍니다`가 정답.** 반복돼도 이를 "남용/완화 대상"으로 보고하지 마라.
  - `~하세요`(너무 가벼움)·단독 `~하십시오`는 **지양** → 행동 안내 문장이 `~하세요`면 그게 area 1 위반(→ `~하시기 바랍니다`로)
  - 단순 평서·기능 설명(`~합니다`, `~할 수 있습니다`)은 그대로 (행동 안내에만 표준 어미 적용)

### 영문 병기 (확정)
- 전문 용어 **첫 등장 1회** `한국어(English)`, 이후 한국어만
- **UI 요소는 한국어 UI 라벨 있으면 한국어만(병기 X)**, 없으면 영어 bold
- → 본문에 영문 UI 라벨 노출(예 "Export DSL")이나 UI 라벨에 불필요한 영문 병기(예 "레코드(Records)")는 area 1

### i18n 라벨 = 권위
- UI 라벨은 `C:\Users\Administrator\Projects\spx-agent\web\i18n\ko-KR\*.json`이 정전. 영문 노출·임의 번역 금지
- 그룹별 주요 i18n 파일은 [[chapter-writing-checklist#4]] §4 또는 그룹 설정 참조

### 직역체 마커 (검색 + 소리내 읽기)
- grep: `을 위한` `를 위한` `로부터` `를 통해` `에 의해` `되어집니다` `상호 배타` `할 수 있게 해` `제공합니다`(무생물 주어) `의 …의`(명사 체인)
- + 소리내 읽어 어색한 무생물 주어, 명사 체인, **음차**(머신 리더블·오케스트레이트 등)

### 피해야 할 패턴
- 기능 중심 도입: "~할 수 있게 해줍니다" → "~할 수 있습니다"
- 군더더기: "~하기 위해서는", "참고로", "유의할 점은" 등 제거
- 없는 기능 안내("~는 운영 차원에서 대응하시기 바랍니다" 류) → 문장 삭제
- 화면에 보이는 라벨·수치 반복(특히 데이터 시각화) 금지

### 콜아웃 용도
- `<Info>` 일반 정보 / `<Tip>` 권장 / `<Note>` 주의 / `<Warning>` 위험. 남용 금지

### 데이터 조회·시각화 챕터 (§C — 대시보드·감사로그·monitor analysis·logs)
- 차트·카드·컬럼 **나열 금지** → 각 요소를 **①의미 ②왜 중요/어떤 질문에 답하나 ③업무 활용** 3단 서술

### 글로서리
- `dataset` → **"지식"** (현재 저장 단위도 "지식"으로 통일). 일반 데이터 의미면 "데이터"
- 노드·UI 라벨은 i18n 검증값 우선 (매개변수 추출기 / Doc 추출기 / List 연산자 등)

---

## 4. findings 4영역 분류 (각 finding은 `rule`로 근거를 cite)

- **area 1 전역 규칙** — 공통 글쓰기 위반. 근거: `E 직역체`/`E 문체`/`E 영문병기`/`E 피해야할패턴` 또는 `A.8 진짜 누락` 등
- **area 2 그룹 규칙** — 그 그룹 한정 사실·구조. 근거: `E 글로서리` 또는 그룹 표기 규칙
- **area 3 전역 규칙 잔존** — Dify/SaaS/Keycloak/Billing 등 남음. 근거: `A.1`~`A.6` 중 해당 번호
- **area 4 확인 필요** — **근거 규칙을 cite할 수 없는 것**(분석본 정합 불일치, 코드·UI 재검증, 규칙 미적용 영역). `rule` 필드엔 "미확인 — 사람 판단 필요"

## 5. 출력 규칙

- 발견 없으면 status `clean`. 사소하면 `minor`. 사실오류/다수면 `issues`.
- ⚠️ **2장(A~D) "이미 결정된 처리"는 절대 findings로 올리지 마라.**
- **모든 finding에 `rule` 필수.** area 1/2/3은 위 A~E의 구체 항목을 cite(예 `A.5`, `B`, `E 문체`). cite 못 하면 **area 1/2/3로 주장하지 말고 area 4**로.
- location=줄 인용/줄번호, issue=무엇이 문제, suggestion=어떻게 고칠지.
- 판단 기준은 "애매한가"(느낌)가 아니라 **"근거 규칙을 cite할 수 있나"**(검증 가능). 문구가 헷갈리면 0장 권위 문서를 Read해 확인 후 cite.
