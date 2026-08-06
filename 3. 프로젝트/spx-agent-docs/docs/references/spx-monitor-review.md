---
type: reference
date: 2026-06-09
last_updated: 2026-06-09
purpose: Monitor 그룹(모니터링·로그·어노테이션 답변) 초안 사용자 리뷰 중 "제품 확인 필요(🔴)" 3건을 spx-agent 소스코드로 직접 조사한 사실 판정 + 문서 수정 방향. 본문 작성/수정 시 참조.
related: [[progress]] §Monitor 그룹 리뷰 피드백, [[1. Daily/2026-06-09]]
---

# Monitor 그룹 리뷰 — 제품 확인(🔴) 코드 조사 결과 (2026-06-09)

> 조사 대상: `C:\Users\Administrator\Projects\spx-agent` (Dify 1.13.3 fork). 읽기 전용 코드 분석.
> 리뷰 피드백 출처: [[1. Daily/2026-06-09]] 메모. 분류 체계: [[progress]] §Monitor 그룹 리뷰 피드백.

---

## ① 어노테이션 답변 — 활성화 경로 (문서 "오케스트레이트 → 기능 추가" 검증)

**판정: 문서 표현은 채팅앱 한정으로만 부분적으로 맞음. 사용자가 본 "로그 및 어노테이션" 탭이 주 경로.**

실제 활성화 경로는 **두 곳**이며 동일 기능의 별개 UI:
1. **(주 경로)** 앱 상세 → **로그 및 어노테이션 탭 → "어노테이션" 하위 탭 상단의 Switch**
   - 라우트: `/app/[appId]/annotations`
   - `web/app/components/app/annotation/index.tsx:156-177` (활성화 Switch)
   - `web/app/components/app/log-annotation/index.tsx:28-61` (Log/Annotation 탭 분기 — **워크플로우 앱은 어노테이션 탭 없음**)
2. **(대체 경로)** 오케스트레이트(Configuration) → 기능 추가 패널의 AnnotationReply 토글
   - `web/app/components/base/features/new-feature-panel/index.tsx:107-108` — **`isChatMode && !inWorkflow` 일 때만 노출** (채팅류 앱 한정)
   - `web/app/components/base/features/new-feature-panel/annotation-reply/index.tsx:69-74`

### ①-1. 앱 유형별 어노테이션 답변 가용성 (대화형 전용)

**어노테이션 답변은 대화형 앱(챗봇·에이전트·채팅 플로우) 전용. 워크플로우·완성형은 사용 불가** (진입점 자체가 없음).

| 앱 유형 (mode)             | 어노테이션 답변 | 설정 위치                             |
| ----------------------- | -------- | --------------------------------- |
| 챗봇 (chat)               | ✅        | 로그&어노테이션 탭 + 오케스트레이트 기능 추가 패널     |
| 에이전트 (agent-chat)       | ✅        | 〃                                 |
| 채팅 플로우 (advanced-chat)  | ✅        | 로그&어노테이션 탭만 (워크플로 에디터라 기능 패널엔 없음) |
| 완성형/텍스트 생성 (completion) | ❌        | 탭 옵션에서 제외                         |
| 워크플로우 (workflow)        | ❌        | 진입점 없음 (탭 슬라이더 미표시 + 기능 패널 제외)    |
|                         |          |                                   |

근거:
- `web/app/components/app/log-annotation/index.tsx:29-31` — `mode === COMPLETION`이면 어노테이션 탭 옵션에서 제외 (로그 탭만)
- `web/app/components/app/log-annotation/index.tsx:47` — `mode === WORKFLOW`이면 탭 슬라이더 자체 미렌더
- `web/app/components/base/features/new-feature-panel/index.tsx:107` — 기능 패널 AnnotationReply는 `!inWorkflow && isChatMode` 조건. 워크플로우·완성형은 `isChatMode=false`라 제외

**문서 수정 방향**:
1. "오케스트레이트 → 기능 추가" 단독 기술은 부정확. **"로그 및 어노테이션 탭에서 어노테이션 응답 토글을 켭니다. (채팅류 앱은 오케스트레이트 설정의 기능 추가 패널에서도 설정 가능)"** 으로 주 경로를 앞세워 수정.
2. 본문 도입부에 **"어노테이션 답변은 대화형 앱(챗봇·에이전트·채팅 플로우)에서 사용하는 기능"** 전제를 한 줄 명시 (워크플로우·완성형 사용자가 헛찾지 않게).

---

## ② 로그 — "개인정보 보호" 섹션 4개 항목 실재 여부

| 항목 | 판정 | 근거 | 문서 방향 |
|------|------|------|----------|
| (a) 보관 기간 설정 | ✅ 실제 기능 (단 env/CLI, UI 아님) | `api/configs/feature/__init__.py:1347-1356` (`WORKFLOW_LOG_RETENTION_DAYS=30`), `api/.env.example:576-583` (`WORKFLOW_LOG_CLEANUP_ENABLED` 등), CLI `clean-expired-messages` (`api/commands/retention.py`) | **구체화**: "서버 환경변수/CLI로 설정하는 관리자 작업"임을 명시. 앱 화면 토글 아님 |
| (b) 로그 익명화 | ❌ 제품 기능 없음 | `anonymiz`/`mask`/`hash` 검색 0건. 로그 조회 시 사용자·세션·이메일 그대로 노출 (`api/controllers/console/app/workflow_app_log.py:82-136`) | **톤다운/삭제**: 제품 기능 아님. 필요 시 "앱/DB 레벨 PII 처리는 운영 책임" 정도로 |
| (c) 접근 제어 | ⚠️ 워크스페이스 RBAC에 종속, 별도 UI 없음 | 로그 조회 = 앱 접근 권한과 동일 (`workflow_app_log.py:70-136` decorator). "로그 전용 접근 설정" UI 없음 | **정확히 명시**: "워크스페이스 권한으로 자동 통제, 별도 로그 접근 설정은 없음" |
| (d) 처리방침 게시·동의 | ❌ 제품 기능 아님 | privacy policy/동의 수집 기능 없음 (admin `privacy_policy` 필드는 탐색 앱 설명용) | **별도 분리**: "운영·법적 고려사항"으로 분리, "제품 기능 아닌 사업자 책임" 명기 |

> ⚠️ 폐쇄망(자체호스팅) 주의: (a)의 메시지 정리 정책 일부(`BillingDisabledPolicy`/`Sandbox`)는 클라우드/빌링 전제 코드. 폐쇄망에서 확실히 동작하는 건 **워크플로 로그 보관(`WORKFLOW_LOG_*` env)** 라인. 본문엔 이 부분 위주로 안내.

**종합 방향**: 현재 "개인정보 보호" 섹션은 (b)(d) 일반론 + (a)(c) 부정확 혼재. → (a)(c)는 실제 동작으로 구체화, (b)(d)는 "운영·법적 고려사항" 별도 박스로 분리해 제품 기능과 구분.

---

## ③ 로그(Logs) — 앱 유형별 표시 항목 차이 (탭 분리 근거)

> 모니터링은 **두 화면**으로 나뉨: ③ **로그(Logs)** 화면 컬럼 차이(아래, 코드 신규 조사) + ③-2 **모니터링(통계/Analysis)** 차트 차이(기존 지식노트). 둘 다 앱 유형별로 다름.

**앱 유형 enum** (`web/types/app.ts:41-48`): `completion` / `workflow` / `chat` / `advanced-chat`(채팅 플로우) / `agent-chat`

**핵심 분기** (`web/app/components/app/log-annotation/index.tsx:47-61`):
- **워크플로우(`workflow`)**: 전용 `WorkflowLog` UI, **어노테이션 탭 없음**
- **그 외(채팅류 + 완성형)**: 공통 `Log` UI. 내부에서 `completion` vs 채팅류 재분기, **`advanced-chat`(채팅 플로우)만 Status 컬럼 추가**

| 화면 요소 | 워크플로우 | 채팅 플로우(advanced-chat) | 일반 채팅/에이전트 | 완성형(completion) |
|---|---|---|---|---|
| 로그 컴포넌트 | WorkflowLog(별도) | 공통 Log | 공통 Log | 공통 Log |
| 테이블 핵심 컬럼 | 시작시간·상태·런타임·토큰·사용자·**트리거 출처** | 요약·사용자·**상태**·메시지수·평가·시간 | 요약·사용자·메시지수·평가·시간 | 입력·사용자·출력·평가·시간 |
| 상세 패널 | 노드별 실행로그 + Replay | 대화 스레드 + 변수 | 대화 스레드 + 변수 | 프롬프트·응답 + 변수 |
| 어노테이션 | ❌ | ✅ | ✅ | ✅ |

근거: `log-annotation/index.tsx:47-61`, `log/index.tsx:74-99`(isChatMode 분기), `log/list.tsx:690-810`(컬럼 조건부), `workflow-log/list.tsx:122-138`(트리거 출처 컬럼).

**문서 구성 제안 (첨부 이미지의 워크플로우/채팅 플로우 2탭 패턴과 정합)**:
- **2탭 권장**: ① **워크플로우 로그** (실행·노드 중심, 트리거 출처·Replay) / ② **대화 로그** (채팅류+완성형, 내부에 채팅 플로우의 Status 컬럼 차이만 짧게 주석)
- 과도하게 5모드 전부 나열하면 복잡 → 사용자 체감 큰 **워크플로우 vs 대화** 2축이 적절.

---

## ③-2 모니터링(통계·Analysis) — 앱 유형별 차트 차이 (기존 지식노트 재활용)

> ⚠️ 새로 조사할 필요 없음 — **이미 볼트에 정리됨**. 모니터링/Analysis 페이지(`/app/[appId]/overview`)는 `appDetail.mode`에 따라 **표시되는 차트 세트 자체가 다름**.
> 출처: [[4. 지식노트/Dify - 앱 통계 UI 구조.md]] (2026-04-24, 차트 표) + [[4. 지식노트/Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름.md]] (2026-04-27, 쿼리 8종/4종 상세)

분기 플래그: `isChatApp`(chat·agent-chat·advanced-chat) / `isWorkflow`(workflow) / completion은 별도 케이스.

| 차트 | completion | chat·agent·채팅 플로우 | workflow |
|---|:--:|:--:|:--:|
| 일별 메시지 수 | | ✓ | |
| 일별 대화 수 | ✓ | ✓ | |
| 일별 사용자 수 | ✓ | ✓ | |
| 대화당 평균 메시지 | | ✓ | |
| 평균 응답 시간 | ✓ | | |
| TPS(초당 토큰) | ✓ | ✓ | |
| 사용자 만족도 | ✓ | ✓ | |
| 토큰 비용 (messages) | ✓ | ✓ | |
| 일별 실행 수 | | | ✓ |
| 일별 터미널 수 | | | ✓ |
| 토큰 비용 (workflow_runs) | | | ✓ |
| 평균 사용자 인터랙션 | | | ✓ |

- 데이터 소스: 메시지 기반 모드 = `statistic.py` 8종(messages 테이블) / 워크플로우 = `workflow_statistic.py` 4종(workflow_runs). 워크플로우는 **금액(total_price) 없음** → 토큰 수만.
- **문서 구성 제안**: 모니터링 챕터의 통계 설명도 **로그와 동일하게 "워크플로우 / 대화" 2탭**으로 차트 세트를 구분. 완성형은 대화 탭 안에서 "평균 응답 시간만 추가, 대화당 평균 메시지 없음" 정도 짧게 주석.

---

## 본문 반영 요약 (Monitor 작성/수정 시)

1. **어노테이션 활성화**: "로그 및 어노테이션 탭" 주 경로로 정정 (+ 채팅앱은 기능 추가 패널도 가능)
2. **로그 개인정보**: (a)보관기간·(c)접근제어 = 실제 동작으로 구체화 / (b)익명화·(d)처리방침 = "운영·법적 고려사항" 별도 분리
3. **모니터링**: 로그 컬럼(③) + 통계 차트(③-2) **둘 다** 앱 유형별로 다름 → 워크플로우 / 대화 2탭으로 구분 표시. 통계 차트는 기존 지식노트([[4. 지식노트/Dify - 앱 통계 UI 구조.md]]) 표 그대로 재활용
