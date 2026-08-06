---
tags: [프로젝트, dify, AI-Agent, HDD, 임시, 마트]
date: 2026-05-11
status: 검토 후 references/mart-design.md 또는 design.md로 이관
related:
  - "[[0. Inbox/대시보드 12종 OLTP vs audit 쿼리 비교]]"
  - "[[3. 프로젝트/spx-agent/references/audit-schema]]"
  - "[[3. 프로젝트/spx-agent/references/dify-db-schema]]"
  - "[[4. 지식노트/Dify - 데이터 소스 3종 비교 (OLTP vs 로그파일 vs 내부 로그 테이블)]]"
---
# 마트 설계 메모 — OLTP/audit 소스 선택 + 드로어 영향

> 목적: 12종 차트 + 드로어/풀뷰까지 한 마트로 묶을 때 고려할 점 정리. 회의 답변 ①(소스 선택) + ②(드로어 확장) 근거 초안.

## 1. 기본 질문 정리 — 90일 캡, audit도 DB인데 왜 다름?

### 화면의 90일 ≠ audit 보존 90일
- **화면의 90일**: 조회 기간 슬라이더 최대 윈도우
- **audit의 90일**: 이벤트가 DB에 남아있는 기간(보존 정책)

dept-cumulative는 "현재 부서별 보유 오브젝트 총량" = **시점 스냅샷(state)**. 누적을 audit으로 재구성하면:
```
누적 = Σ(create 이벤트) − Σ(delete 이벤트)  ← 처음부터 지금까지
```
91일 전 만든 앱은 `app_create`가 이미 사라짐. 그 앱이 오늘 삭제되면 `app_delete`만 남아 **누적 음수**. 화면 윈도우랑 무관하게 망함.

→ **시점 스냅샷은 OLTP `resource_ownership` 1번 SELECT가 정답.** audit 보존 늘려도 "처음부터"가 아닌 한 못 따라잡음.

### OLTP vs audit — 둘 다 DB지만 본질이 다름

| 구분 | OLTP (resource_ownership, messages) | audit (audit_events) |
|---|---|---|
| 본질 | **상태(state)** — 지금 이렇다 | **이벤트(event)** — 이때 이런 일이 있었다 |
| 갱신 | mutable (행 추가/변경/삭제) | append-only, 불변 |
| 삭제 처리 | 행이 사라짐 | `_delete` 이벤트가 추가 |
| 보존 | 거의 영구 | 90일 |
| 컬럼 | 정규 타입 컬럼 | 대부분 `details jsonb` |

audit는 OLTP를 5분 폴링한 **이벤트 스트림 사본**. 거울이 아님.
- 현재 상태 → OLTP
- 변경 이력 → audit
- messages 같은 append-only 트랜잭션 데이터는 둘 다 가능

## 2. 둘 다 가능할 때 선택 우선순위

1. **단일 진실 원천(SSOT)** — 같은 화면 안에서 차트별로 소스 다르면 숫자 불일치. 메트릭군 단위로 통일
2. **의미 일치** — OLTP=앱 소유 부서, audit=호출자 소속 부서. 우리 화면은 전자
3. **비용** — jsonb 추출 vs 정규 컬럼, OLTP 2~5배 우세
4. **정규화 품질** — audit이 이기는 유일 케이스(top-error-types `action`)

→ 둘 다 가능 = OLTP 채택이 기본값. 현 12종 결정(OLTP 11 + audit 1) 그대로 유효.

## 3. 드로어 화면 추가 — Grain이 바뀜

차트는 집계(aggregate), 드로어는 이벤트 단위(detail row). 마트 grain이 달라짐.

| | 차트 12종 | 드로어 |
|---|---|---|
| 단위 | 부서별 합계, RPS | 개별 이벤트 1건 |
| 행 의미 | "재무 14.5K" | "최영재가 14:32에 재무 Q&A에 질문" |
| 컬럼 | dept, count, rate | timestamp, user, app, message, status, error |
| 정합성 | 부서 합계 = KPI | **드로어 6건이 차트 14.5K의 부분집합이어야** |

마지막 줄이 결정적. 차트=OLTP / 드로어=audit이면 5분 폴링 지연 때문에 차트엔 잡혔는데 드로어엔 없는 호출이 생김.

### 드로어 영역에서 audit이 부각되는 이유
1. **정규화 액션명** — message_send / workflow_execute / app_create 등을 한 화면에 섞기 쉬움. OLTP면 messages UNION workflow_runs UNION resource_ownership 다 합쳐야 함
2. **에러 분류** — "model_timeout — 30s 초과" 같은 라벨은 audit action 기반
3. **풀뷰 = audit 페이지** — "전체 보기 ↗"가 `/audit`. 드로어가 OLTP면 미리보기 ↔ 풀뷰 불일치 가능

### 그러나 audit엔 메시지 본문 없음 ⚠️
드로어가 보여주는 "월 결산 자료 요약 요청" 같은 본문 — `messages.ts` collector details 페이로드:
```js
{ messageId, conversationId, modelProvider, modelId, 
  messageTokens, ..., error, fromSource, invokeFrom, workflowRunId }
```
**query 본문이 없음.** audit 단독으로 미리보기 못 만듦. messageId → OLTP `messages.query` JOIN 필수.

확인 필요:
- [ ] `/audit` 페이지가 message query 본문을 어떻게 표시하나 (이미 JOIN 하고 있을 수도)
- [ ] 안 하면 collector `messages.ts`에 `queryPreview` 추가 요청 (200자 trim, `prompt-changes.ts` 선례)

## 4. 권장 마트 구조 (2-layer)

```
┌──────────────────────────────────────────────────────────┐
│  Gold (집계 fact) — 차트 12종용                          │
│    fact_calls_daily   (date, dept_id, app_id, model_id) │
│    fact_errors_daily                                     │
│    fact_users_daily                                      │
│    snap_resources_daily (SCD2 — dept-cumulative용)      │
└──────────────────────────────────────────────────────────┘
          ↑ roll up
┌──────────────────────────────────────────────────────────┐
│  Silver (이벤트 fact) — 드로어/풀뷰용                    │
│    fact_event                                            │
│      occurred_at, actor_id, actor_name, dept_id,         │
│      app_id, app_name, action, status, error_type,       │
│      message_preview, model_id, source_system            │
└──────────────────────────────────────────────────────────┘
          ↑ ETL
┌──────────────────────────────────────────────────────────┐
│  Bronze/Staging                                          │
│    OLTP messages, workflow_runs, accounts, resource_own. │
│    audit.audit_events                                    │
└──────────────────────────────────────────────────────────┘
```

### 원칙
1. **드로어와 차트가 같은 silver fact_event에서 파생** — 정합성 자동. 차트는 roll up, 드로어는 직접 조회
2. **fact_event는 OLTP 우선 + audit 보강**:
   - 본문/타임스탬프/사용자/모델/토큰 → OLTP messages·workflow_runs
   - error_type 정규화 라벨 → audit action JOIN
   - dept_id 매핑 → OLTP resource_ownership (앱 소유 부서)
3. **source_system 컬럼** — "OLTP에서 1차 추출 + audit으로 라벨 보강" 추적
4. **ETL 주기** — OLTP 실시간 vs audit 5분 폴링. fact_event 적재 주기를 audit 폴링에 맞춰 정렬해야 정합. 차트만 OLTP 실시간 / 드로어 5분 지연 명시 등 정책 결정 필요

## 5. 새로 추가된 고려사항 5개

| # | 고려사항 | 결정 필요 |
|---|---|---|
| 1 | 차트↔드로어 정합성 | 같은 silver fact에서 파생 (마트 2-layer) |
| 2 | 통합 활동 로그 단순화 | audit `action` 분류 라벨 활용 |
| 3 | 메시지 본문 출처 | OLTP messages JOIN 필수 / collector 보강 요청 |
| 4 | 폴링 지연 vs 실시간 | "5분 전까지 반영" 명시 or OLTP 실시간 |
| 5 | 부서 매핑 의미 통일 | 앱 소유 부서 기준으로 드로어 필터 |

## 6. 부서 매핑 함정 (드로어에서 재발)

"재무팀 활동" 카피 — 차트의 "재무 14.5K"는 **재무팀 소유 앱의 호출량**, 드로어 행 "최영재(재무)"는 **호출한 사람의 소속**. 두 개가 다를 수 있음.

예: 마케팅팀 직원이 재무 Q&A 앱 호출 → 차트 "재무 14.5K"엔 잡힘 → 드로어 필터가 actor 부서=재무면 빠짐 → 6건 ≠ 14.5K 부분집합.

→ 드로어를 **앱 소유 부서 기준**으로 필터해야 차트와 맞음. 행 표시는 "최영재(소속: 마케팅)"로 둘 다 노출하거나, "재무 앱 사용자" 관점으로 통일.

## 7. 미해결 / 후속

- [ ] `/audit` 페이지 message query 본문 표시 방식 확인 (이미 JOIN 처리?)
- [ ] `messages.ts` collector에 `queryPreview` 200자 trim 필드 추가 요청 검토
- [ ] fact_event ETL 주기 결정 — OLTP 폴링 주기 vs audit 5분 정렬 정책
- [ ] 드로어 부서 라벨 UX 결정 — "앱 소유 부서" vs "actor 소속 부서" 표시법
- [ ] 마트 1차 DDL 초안 — fact_event, fact_calls_daily, snap_resources_daily
- [ ] dept-error-table "주원인" 컬럼은 마트에서 자동 보강 (silver의 error_type 활용)
- [ ] 검토 후 `references/mart-design.md`로 승격 또는 `hdd/design.md § 마트` 섹션 추가
