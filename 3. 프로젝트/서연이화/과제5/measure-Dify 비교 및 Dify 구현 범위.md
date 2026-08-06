---
tags:
  - 서연이화
  - 과제5
  - 대책서
  - dify
  - RAG
  - LLM
created: 2026-06-26
source_measure: C:\Users\Administrator\Projects\seoyoneh\tmp\measure
source_dify: 3. 프로젝트/서연이화/과제5/대책서 작성 도우미 v0.0.2.yml
---

# measure ↔ Dify 비교 및 Dify 구현 범위 정의

> [!summary] 한 줄 결론
> **Dify는 "stateless 추천 생성기"만 담당한다.** measure의 복잡함 대부분은 AI 추론이 아니라 *상태 관리(세션·선택·누적·원인별 묶음·DB 저장)* 이고, 이건 **호출하는 백엔드(자사 플랫폼/S-CLM)의 책임**이다. Dify 파일이 간단해 보이는 건 틀린 게 아니라 **범위를 올바르게 좁힌 결과**다. 단, 정작 과제5의 본질인 **실제 RAG 검색이 아직 더미 코드**라는 점만 채우면 된다.

관련 문서: [[3. 프로젝트/서연이화/과제5/대책서 작성 비즈니스 로직.md]] · [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 버전 기록.md]] · [[3. 프로젝트/서연이화/과제5/무제.md]]

---

## 1. 날짜 · 버전 매핑

| 날짜 | 버전 | 대상 | 내용 |
|---|---|---|---|
| 2026-06-25 | measure 소스 | `com.measure` (Spring Boot) | 현상→원인→대책 단계 작성 **레퍼런스 구현**(상태·세션·DB 포함) |
| 2026-06-25 | dify v0.0.1 | [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 v0.0.1.yml]] | 외부 LLM 단일, 문자열 4개, 점수 없음 |
| 2026-06-26 | dify v0.0.2 | [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 v0.0.2.yml]] | 내부(더미 RAG)+외부 혼합, 점수, {text,score}, 병합·정렬 **(현재 작업본)** |
| 2026-06-26 | **본 분석** | 이 노트 | measure ↔ dify v0.0.2 비교, Dify 구현 범위 확정, 워크플로우 옵션 A/B/C |
| (예정) | dify v0.0.3 | — | 더미 코드노드 → **Knowledge Retrieval(실제 RAG)** 교체, 모델명 실교체 |

---

## 2. measure(레퍼런스) 전체 로직

```
[현상 입력] create()  → 문서 1건 = ChatGPT 세션 1개 (previous_response_id로 맥락 유지)
   ↓
[원인 추천] suggestCauses()   내부(더미RAG N) + 외부(LLM N), 기본 5/5·합≤10
   ↓  "나머지 계속 찾기" = regenerate + keep(체크유지) + exclude 누적, round++
[원인 선택] setCause(List)    여러 개 선택 → 번호 묶음, selected=true
   ↓
[대책 추천] suggestCountermeasures(causeId)
            선택된 원인 루프 → 원인마다 LLM 1회 호출 → parentCauseId 귀속
   ↓  원인별 "대책 더 찾기" = causeId + keepIds (그 원인 안에서만 재롤)
[대책 선택] setCountermeasure(causeId, ids, manual, notes)  원인 범위만 갱신, MANUAL 직접입력
   ↓
[작성완료] complete() → openaiSessionActive=false (세션 종료)
```

복잡함의 정체 = **상태가 DB에 산다**: `Suggestion` 엔티티에 `selected / discarded / round / parentCauseId / note / ragSource / score / source(INTERNAL·EXTERNAL·MANUAL)` 가 전부 저장됨. 내부는 `InternalKnowledgeService` 더미(풀 20개 + 의사 유사도 점수 + 더미 RAG 근거).

---

## 3. measure ↔ Dify v0.0.2 비교표

| 항목 | measure | Dify v0.0.2 | 누가 담당 |
|---|---|---|---|
| 추천 생성(내부+외부 혼합, 점수) | O | **O (동일)** | Dify |
| 개수 보정 5/5·≤10 | O | O (파싱 코드노드) | Dify |
| exclude 재추천 | O (서버 누적) | △ (호출자가 exclude 주입) | 백엔드 |
| 세션 맥락(previous_response_id) | O | **X (workflow는 stateless)** | 백엔드 |
| 선택/확정/discarded 상태 | O (DB) | **X** | 백엔드 |
| 원인별 대책 묶음 | O (parentCauseId) | △ (원인 1개씩 호출, 묶음은 호출자) | 백엔드 |
| 실제 RAG 검색(7천건 대책서) | 더미 | **더미 (코드 하드코딩)** | Dify(예정) |

> [!note] 핵심
> Dify에는 **"AI가 추천을 만든다"** 만 남고, 나머지(상태·세션·선택·누적·묶음·저장)는 전부 호출자(백엔드)로 빠졌다. 그래서 간단하다. **정상이고 올바른 분리다.**

---

## 4. Dify 구현 범위 정의 (★ 이 노트의 핵심)

**역할 분담 — Dify는 어디까지만 한다**

| 책임 | 담당 | 이유 |
|---|---|---|
| AI 추론 (현상→원인, 원인→대책 생성) | **Dify** | LLM·RAG 호출이 본질 |
| 지식 검색 (과거 대책서 7천건) | **Dify (Knowledge 노드, 예정)** | RAG = Dify가 잘하는 영역 |
| 상태·세션·선택·누적·문서 저장 | **백엔드(자사 플랫폼/S-CLM)** | DB·트랜잭션·동시작성 = 앱 책임 |
| 화면·선택 UI·원인별 버튼 | **프론트** | measure의 app.js 영역 |

### 범위 관련 결론 3가지
- **API(워크플로우) 개수**: 추천 생성용으로 **1개로 충분**. (분리는 아래 옵션 B 참고)
- **DB 저장**: **Dify 안에는 불필요.** stateless 유지가 정석. 문서·선택·세션·완성 대책서 저장은 백엔드 DB. ("과거 품질이력 DB"는 Dify가 *읽는* RAG 소스일 뿐, *쓰는* 곳이 아님)
- **지금 범위면 되나**: 추천 생성 한정으로는 **적절**. 단 **실제 RAG(지식베이스) 연동**이 빠진 채 더미 코드라는 점만 채우면 됨 → 다음 작업 1순위.

---

## 4.5. 입출력 형식 대조 (measure API ↔ Dify) — ⚠️ 소스에 맞춰 dify 필드 수정 필요

> [!warning] 결론
> **동일하지 않다.** 차이는 두 종류 — (1) **정상**: 레이어/상태 차이(상태 필드·현상 조립은 백엔드 몫), (2) **고쳐야 함**: `text↔content` 키 이름, source 대소문자, 응답 메타데이터 부재. **dify 파일은 지금 수정하지 않고**, 아래 매핑을 백엔드 어댑터/계약에 반영하거나 추후 dify 정리 시 참고.

### 입력 대조
| | measure (추천 생성 요청) | Dify (payload) |
|---|---|---|
| 식별 | `{id}` (path, 문서 id) | 없음 (stateless) |
| 내용(현상·원인·exclude) | **DB에서 읽음** | **payload로 직접 전달** |
| 분기 | 엔드포인트 2개로 구분 | `step` 필드 |
| 제어값 | `regenerate, model, internalCount, externalCount, causeId, keep[], keepIds[]` | `internalCount, externalCount` |

→ 공통은 `internalCount/externalCount`뿐. measure는 "상태 참조(id)+제어값", Dify는 "내용 직접 전달". **stateless vs stateful의 당연한 차이** (정상).

### 출력 대조 — Dify item은 measure *API 출력*이 아니라 한 단계 안쪽 *내부 ScoredItem*에 가까움
| 항목 | 내부 `ScoredItem` | **API `SuggestionView`** | Dify item |
|---|---|---|---|
| 문구 | `text` | **`content`** | `text` |
| 점수 | `score` | `score` | `score` |
| 출처 | (별도 enum) | `source` = **INTERNAL/EXTERNAL** (대문자) | `source` = **internal/external** (소문자) |
| RAG근거 | `ragSource` | `ragSource` | `ragSource` |
| id / selected / parentCauseId / round / note | 없음 | **있음** | 없음 |

응답 래퍼: measure `SuggestResponse = { round, items[], internalAdded, externalAdded, model, totalTokens, cost, docTotalTokens, docTotalCost }` vs Dify `result = { items[] }` 뿐.

### ★ 소스 기준 매핑 규칙 (dify 산출물 → measure 모델)
1. **`text` → `content`** : 키 이름 변경 (measure API는 `content`).
2. **`source` 대문자화** : `internal/external` → `INTERNAL/EXTERNAL`. (`MANUAL`은 dify 산출물 아님 — 백엔드의 직접입력 처리)
3. **상태 필드는 백엔드가 부여** : `id/selected/parentCauseId/round/note` — stateless dify가 못 만듦 (정상, 변경 불필요).
4. **응답 메타데이터 추가 권장** : `round/internalAdded/externalAdded/model` 등을 dify result에도 실으면 화면 표기·디버깅에 바로 쓰임 → 6.5절 2번과 연결.

### 커버리지 — Dify는 measure 엔드포인트의 일부만
| Dify | measure 대응 |
|---|---|
| step=cause | `POST /{id}/cause-suggestions` |
| step=countermeasure | `POST /{id}/countermeasure-suggestions` |
| — | `POST /documents`(생성) · `POST /{id}/cause`(원인확정) · `POST /{id}/countermeasure`(대책확정) · `POST /{id}/complete`(완료) · `GET 목록/상세/models` |

→ Dify는 **추천 생성 2개만** 담당, 나머지는 상태 연산이라 백엔드 전용. 범위 정의(4절)와 일관.

---

## 5. 워크플로우 구성 옵션 A / B / C 비교

| 구분 | A. 단일 워크플로우 (현재 v0.0.2) | B. 원인/대책 2개 분리 | C. 대책에 Iteration 추가 |
|---|---|---|---|
| 구조 | 1개 + `step`으로 원인/대책 분기 | 원인용·대책용 워크플로우 각각 | A 또는 B + 대책 노드에 Iteration |
| 원인별 대책 처리 | 호출자가 원인 수만큼 N회 호출 | 호출자가 원인 수만큼 N회 호출 | 원인 목록 1회 호출 → 일괄 생성 |
| **장점** | 가장 단순, 파일 1개로 관리, 빠른 시작 | 프롬프트·모델·지식베이스 **독립 튜닝**, 책임 명확 | 호출 횟수↓, 호출자 루프 부담↓ |
| **단점** | 분기 코드/지식소스가 한 파일에 섞임, 튜닝 충돌 | 파일 2개 관리, 공통 코드 중복 | 부분 실패·재시도·원인별 재롤 제어 어려움, 복잡도↑ |
| 적합 시점 | **지금(초기·PoC)** | 운영 안정화 단계 | 원인 수가 많고 지연이 문제될 때 |
| 권장 | ✅ 현재 채택 | 다음 단계 후보 | 성능 이슈 발생 시 |

> [!tip] 권장 경로
> **A(지금) → RAG 연동(v0.0.3) → 필요 시 B로 분리 → 지연 이슈 시 C(Iteration)** 순으로 단계적 확장.

---

## 6. Dify v0.0.2의 문제점 & 해결 (설계 단계 우선순위)

> [!warning] 분류 기준
> 지금은 **운영이 아니라 구조 설계 단계**다. 그래서 기준은 "지금 데이터가 맞나"가 아니라 **나중에 바꾸기 비싼 결정(아키텍처 경계·입출력 계약)을 먼저 잡았는가**다. 실제 지식(RAG) 적재처럼 운영 때 채우면 되는 건 뒤로 내린다. 반대로 "알면 되는 한계"로 보였던 것들이 사실 **구조 결정**이라 앞으로 온다.

### 🔴 지금 결정 — 나중에 바꾸기 비싼 아키텍처/계약
1. **상태·세션 책임 경계 (workflow vs chatflow)** — workflow 모드는 맥락 메모리가 없어 measure의 `previous_response_id` 맥락을 백엔드가 exclude로만 재현한다.
   - 설계 결정: "Dify는 stateless 추천기, 상태는 백엔드" 를 **확정**할지, 아니면 chatflow + conversation 변수로 맥락을 Dify에 둘지. 이건 나중에 갈아엎기 가장 비싼 결정.
   - 권장: **stateless 확정**(RAG 기반에선 exclude 방식이 더 깔끔). 단 문서에 경계를 못박아 둘 것.
2. **입출력 계약(payload / result 스키마) 고정** — `step / phenomenon / cause / internalCount / externalCount / exclude` ↔ `items:[{text, score, source, ragSource}]`.
   - 설계 결정: 백엔드와 이 계약을 **버전 고정**. 필드 추가는 호환되게(옵셔널). 계약이 흔들리면 프론트·백엔드가 같이 깨짐.
3. **점수(score) 정책** — 내부=벡터 유사도, 외부=LLM 자기평가(임의값). 한 척도로 섞어 정렬하면 왜곡.
   - 설계 결정: 외부 score 제거(null) 또는 **Rerank 노드로 통일** 중 택1 → 병합·정렬 노드와 출력 계약에 직접 영향. 점수의 "의미"를 지금 정의해 둘 것.

> [!note] 설계 메모 — 맥락 기억을 어디서 할까 (1번 결정의 근거)
> **"맥락 기억"은 두 종류로 갈라야 한다.**
> - **① LLM 대화 맥락** — `previous_response_id`로 잇는 직전 턴 기억(톤·중복회피). LLM 추론 보조용.
> - **② 애플리케이션 상태** — 문서·선택·round·원인↔대책 귀속·완성 대책서. 비즈니스 데이터.
> "상태를 백엔드에서"는 ②, "Dify에서 맥락 기억"은 보통 ①(chatflow + conversation 변수). 따로 판단.
>
> **Dify에 맥락 기억을 두면 좋은 경우** — (a) 멀티턴 자연어 UX("아까 그거 말고")가 핵심 가치, (b) 페이로드를 `conversation_id` 하나로 경량화, (c) 프로토타입 속도.
> **굳이 안 해도 되는 경우(=지금 과제)** — 추천이 **stateless 함수**라 출력이 `(현상+원인+exclude+RAG결과)`로 결정됨. 대화 히스토리가 결과를 안 바꾼다. 중복회피는 `exclude` 명시로 충분하고, RAG가 들어오면 다양성·관련성은 검색이 책임 → 메모리 역할이 거의 사라짐.
>
> **백엔드 상태 소유(stateless Dify) vs Dify 맥락 보유 차이**
> - 트랜잭션·권한·감사로그: DB는 보장, Dify conversation 변수는 **보장 없음**.
> - 여러 문서 동시작성·격리: 백엔드는 문서별 관리가 자연스럽지만, Dify는 conversation 단위라 문서 매핑을 백엔드가 또 해야 함 → **이중 관리**.
> - 재현성·이식성: 입력이 명시적이면 추적·교체 쉬움. Dify 내부 메모리에 숨으면 어려움 + Dify 버전 종속.
>
> **결론**: measure의 진짜 상태(②)는 트랜잭션·격리·감사가 필요한 비즈니스 데이터라 **DB 소유여야 함.** ②가 백엔드에 있으면 ①도 백엔드가 exclude로 재현하는 게 일관됨 → **둘 다 백엔드, Dify는 stateless.** 감수할 비용은 exclude 누적으로 페이로드가 커지는 것 하나(최근 N개만 exclude + 나머지는 RAG 메타데이터 필터로 관리). chatflow는 "대화형 챗봇", workflow는 "추천 생성 API(함수)" — 지금은 후자라 workflow가 결이 맞다.

### 🟠 구조 반영 — 노드/분기 추가 (곧)
4. **RAG 교체 지점을 인터페이스로 비워두기** — 지금 더미 코드노드가 `items:[{text,score,source,ragSource}]` 를 그대로 내보내게 돼 있음. 이 **출력 시그니처를 고정**해 두면 나중에 코드노드 → Knowledge Retrieval 노드로 바꿔도 뒷단이 안 깨짐.
   - 설계 포인트: 실제 지식 적재는 운영 때(아래 🟡) 하더라도, **"여기가 교체 지점"** 이라는 걸 지금 구조에 명시.
5. **개수 강제 불가** — Dify 정적 스키마라 `minItems/maxItems`를 못 박음. "정확히 N개" 프롬프트 + 병합 노드 slice 보정에 의존.
   - 설계 포인트: 출력 개수 일관성을 **병합 노드가 책임지게** 구조화(프롬프트 준수에만 기대지 말 것).
6. **`external_count=0` 분기 없음** — 내부만 원해도 외부 LLM 노드가 실행됨. IF/ELSE 한 단 추가 = 그래프 변경.

### 🟡 운영 단계 — 나중에 채워도 됨
7. **실제 RAG 지식 적재** — 코드 더미 20개 → 과거 대책서 7천건 지식베이스. **지금은 구조 설계 단계라 미뤄도 됨**(인터페이스만 4번처럼 잡아두면 교체는 쉬움).
8. **코드 중복 2벌** — 내부 원인/대책·병합 노드가 거의 같은 코드 2벌 → 유지보수 시점에 공통화 검토.

---

## 6.5. 구조 설계 · 리팩토링 추가 포인트 (운영 데이터 없이 지금 손볼 것)

6번이 "결정/우선순위"라면, 여기는 **지금 손대도 되는 구체적 리팩토링** 목록이다.

1. **에러·가드 응답 표준화** — 현재 파싱 코드가 `raise Exception('현상이...')` 로 죽음. 워크플로우가 예외로 끝나면 백엔드는 HTTP 5xx만 받아 사유를 못 씀.
   → 가드 실패를 예외 대신 **정상 출력**(`result:{ ok:false, errorCode, message, items:[] }`)으로 내보내 백엔드가 파싱·표시하게. 출력 계약(2번)과 한 세트.
2. **응답 메타데이터 추가** — 지금 `result.items` 만 나옴. measure의 `SuggestResponse` 처럼 `usedModel / internalCount / externalCount / round` 같은 메타를 같이 내보내면 백엔드 디버깅·화면 표기(내부5/외부5 등)에 바로 쓰임.
3. **모델을 payload 변수로 노출** — 지금 노드에 `gpt-5-nano` 고정. 모델 변경·A/B 테스트하려면 `payload.model` 로 받아 노드에 주입. (모델명 자체는 문제 없음, 하드코딩이 문제)
4. **step을 확장 가능하게** — 현재 `cause / countermeasure` 2값 IF/ELSE. 향후 "점검 가이드 생성" 등 단계 추가 여지를 두고 분기 구조를 설계(2분기 고정에 묶이지 않게).
5. **exclude 정규화 위치 일원화** — 지금 trim/중복제거가 파싱·내부풀·병합 **3군데** 흩어짐. 파싱 노드 한 곳에서 정규화하고 이후는 신뢰하는 구조로.
6. **노드 식별자 가독성** — `'2002000000001'` 같은 숫자 ID는 유지보수 시 추적이 어려움. Dify가 허용하는 선에서 desc/title로 의미를 분명히(편집 시 혼동 방지).
7. **중복 코드 처리 전략 확정** — 내부풀·병합 코드가 원인/대책 2벌. **옵션 B(원인/대책 워크플로우 분리)로 가면 자연 해소**되므로, 5번(워크플로우 구성)과 묶어서 결정.

> [!tip] 설계 단계 권장 순서
> (1) 상태 경계 stateless 확정 → (2) 입출력 계약·에러 응답 표준화 → (3) 점수 정책 결정 → (4) RAG 교체 지점 인터페이스 고정 → (그 후 운영 단계에서) 실제 지식 적재.

---

## 6.6. 작업 순서 결정 — "RAG 임의 테스트" vs "구조 설계" (2026-06-26)

> [!question] 고민
> RAG 문서가 없어 못 하는데, **임의 생성해 지금 테스트**하는 게 나을지 **구조 설계/리팩토링**을 보는 게 나을지. 이유: JSON 필드 수정도, 병합·정렬도 **지식 검색 output 모양에 좌우**됨.

### 결론: "둘 중 하나"가 아니라 **순서를 나눈다**. 일을 RAG 종속/독립으로 쪼갠다.

| RAG output에 **독립** (지금 확정 가능) | RAG output에 **종속** (모양 알아야 확정) |
|---|---|
| 상태 경계(stateless) 결정 | 내부 item 필드명 (content/score/metadata?) |
| 입력 계약(payload: 현상·원인·exclude·counts·step) | 병합·정렬 (점수 척도 통일) |
| 에러·가드 응답 표준화 | source/ragSource 형태 |
| 워크플로우 A/B/C 결정 | 출력 계약의 item 부분, text↔content 최종 매핑 |
| 외부 LLM 분기 | |

→ 왼쪽은 **지금** 박아도 안 흔들림. 오른쪽은 **RAG 모양을 한 번 본 뒤** 확정.

### 핵심: 그 "모양"은 풀 RAG 없이도 얻는다 — 목적을 *품질 테스트*가 아니라 *모양 탐색(spike)*으로 좁혀라
- 필요한 건 7천건·청킹·rerank 튜닝이 아니라 **Knowledge Retrieval 노드의 출력 스키마 1개**.
- 그건 **더미 대책서 10~30개**만 올려도 동일하게 나옴 → 등록 → 노드 1회 실행 → output 캡처 → 필드·점수 척도 확정.
- **즉, 목적을 spike로 좁히면 "지금 임의 생성"이 맞다.** (품질 검증용 풀 테스트는 뒤로)

### ⚠️ 더미 만들 때 미리 갈리는 숨은 설계 — 검색결과를 후보로 바꾸는 방식
현재 더미는 완성된 "원인/대책 문구"를 바로 반환. 실제 Retrieval 노드는 **검색된 과거 대책서 청크**(`content`+`metadata`+검색 `score`)를 반환 → "검색 결과 → 원인/대책 후보" 변환이 필요:
- **(A) 구조화 레코드형** — 지식베이스를 `현상/원인/대책` 구조로 적재 → metadata에서 원인·대책 직접 추출. (과제: "DB 스키마 명확히 구조화")
- **(B) 청크→LLM 추출형** — 검색 원문 청크를 LLM에 넘겨 후보로 가공.

→ A/B가 **output 모양·병합·필드명을 다르게** 만듦. 더미 spike 때 A/B를 결정하면 오른쪽 칸 설계가 풀림.

### 추천 순서
1. **지금**: RAG 독립 항목 확정 → 체크리스트 별도 관리: [[3. 프로젝트/서연이화/과제5/RAG 독립 항목 확정 체크리스트.md]] (A. 완전독립 9개 / B. 부분종속 2개 / C. 여지 1개)
2. **다음 (작은 spike)**: 더미 10~30개로 Retrieval 노드 1회 → 출력 스키마·점수 척도 캡처 + A/B 결정.
3. **그 후**: 그 모양 기준으로 필드명·병합·정렬·출력 계약 확정 (4.5절 매핑 규칙을 실제 RAG 키로 갱신).
4. **운영 단계**: 7천건 적재 + 청킹·rerank 품질 튜닝.

> 한 줄: **풀 RAG 품질 테스트는 뒤로, RAG 출력 모양 spike는 지금.** 모양 모른 채 필드/병합 확정 금지, 모양 알려고 풀 RAG 대기도 금지.

---

## 7. 발전 방안 (운영 환경 고려)

- **RAG 고도화**: vector top-k 넘어 **메타데이터 필터**(품명·이슈타입·차종) + **Rerank**. (과제 요약: "이슈타입/품명별 검색 볼륨 분리", "청킹 룰이 관건")
- **원인별 대책 Iteration**: 옵션 C 참고.
- **관찰성**: 토큰/비용/실패율 로깅(measure엔 `SessionLog`·비용계산 있음).
- **신뢰성**: rate limit, 모델 fallback, 타임아웃, 프롬프트 버전 관리.
- **사람 개입(HITL)**: 과제정의서 "AI는 방향(점검항목) 제시, 사람이 수행 완성" → 대책서는 **초안 + 점검 가이드**까지만, 확정은 사람. 현재 "점검 가이드 생성" 단계는 Dify에 없음 → 추가 후보.

---

## 8. 관련 노트
- [[3. 프로젝트/서연이화/과제5/대책서 작성 비즈니스 로직.md]]
- [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 버전 기록.md]]
- [[3. 프로젝트/서연이화/과제5/무제.md]]
- [[3. 프로젝트/서연이화/서연이화 과제 요약.md]]
