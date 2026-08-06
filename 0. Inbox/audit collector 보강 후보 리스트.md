---
tags: [프로젝트, dify, AI-Agent, audit, collector]
date: 2026-05-12
status: 1차 후보 리스트 — 마트 ETL 작업하면서 부족분 누적 → 완료 후 최종본 이사님 보고
related:
  - "[[1. Daily/2026-05-12.md]]"
  - "[[3. 프로젝트/spx-agent/references/audit-schema.md]]"
  - "[[0. Inbox/Dify의 에러 처리 흐름 추적 산출물.md]]"
purpose: audit collector(별도 dify-audit 컨테이너)가 audit_events.details에 박아야 할 필드 추가 요청 후보. 이사님 OK — 사용자 직접 작업 가능, 단 리스트 최종본은 이사님께 보고.
---

# audit collector 보강 후보 리스트

> **이사님 OK**: audit 테이블(`audit_events.details`)에 데이터 추가하는 거 사용자가 직접 작업 가능.
> 단 리스트 최종본은 이사님께 보고 필요.
>
> **본 노트의 역할**: 마트 ETL 작업하면서 부족분이 추가될 수 있으니, 누적 리스트로 관리. 최종 확정 시점에 이사님 보고용 정리본 만듦.

## 컨텍스트 — 왜 보강이 필요한가

2026-05-12 오후 이사님 미팅으로 **마트 = audit 단독 + RBAC JOIN** 확정. 마트 ETL 입력이 모두 `audit_events`에서 옴.

근데 5/8 노트에 박힌 발견: **`workflow-runs.ts` collector는 5필드만 박아서 비대칭**. messages.ts(14필드)와 워크플로우 정보 정합성 없음. 이게 audit 단독 마트의 진짜 차단 요소.

마트 ETL이 H-DASH-01(이중카운트), H-DASH-03(디버깅 필터), H-DASH-18(에러 분류) 방어하려면 collector가 충분한 필드를 박아야 함.

## 1차 후보 리스트 (오늘 시점)

### `workflow-runs.ts` (현재 5필드만 — 보강 우선순위 최고)

| 필드 | OLTP 원본 | 마트 활용 | 우선순위 | 근거 |
|---|---|---|---|---|
| `triggered_from` | `workflow_runs.triggered_from` | 디버깅 필터 (`= 'app-run'` 만 집계) | ⭐⭐⭐ 필수 | H-DASH-03 방어 (mock에 디버깅 데이터 섞여있으면 차트 왜곡) |
| `invoke_from` | `workflow_runs` 직접 컬럼 없음 — 추정 | 디버깅 필터 보완 | ⭐⭐ | H-DASH-03 (단 워크플로우엔 invoke_from 없을 수 있음 — 확인 필요) |
| `total_price` | `workflow_runs.total_price` | 비용 추론 | ⭐⭐ | 모델별 비용 분석 기반 |
| `error` | `workflow_runs.error` (LongText) | H-DASH-18 에러 분류 (ILIKE 룰) | ⭐⭐⭐ 필수 | 현재 workflow 에러는 audit details에 없어 분류 불가 — 옵션 D 적용 못함 |
| `model_provider` / `model_id` | `workflow_node_executions.process_data` (jsonb 파싱 필요) | 모델별 토큰/호출 분포 | ⭐⭐ | 마트 ETL이 직접 jsonb 파싱 회피 — collector가 노드 분해해서 박아주면 가장 깔끔 |

### `messages.ts` (현재 14필드 풍부 — 1개만 추가 요청)

| 필드 | OLTP 원본 | 마트 활용 | 우선순위 | 근거 |
|---|---|---|---|---|
| `queryPreview` (200자 trim) | `messages.query` | 차트 드로어 본문 표시 | ⭐⭐ | audit details에 query 본문 없음 → OLTP JOIN 회피하려면 collector에 trim 본문 추가 필요. 5/11 마트 메모 § 3 발견 |

### 기타 collector (확인 필요)

- `workflow-nodes.ts` (5/8 노트엔 `triggeredFrom` 박혀있다 함 — 비대칭 발견)
- `app.ts`, `account.ts`, `auth.ts` 등 13종 collector
- → audit details 필드 명세 검증 (우선순위 2.5 작업)에서 추가 발견 가능

## 마트 ETL 작업하면서 추가될 수 있는 후보

> ETL 작성 중 "audit details에 이 키가 있어야 분류 가능한데 없음" 발견할 때마다 여기에 추가

### 누적 후보 (TBD)
- [ ] (마트 ETL 작업 시 발견되는 부족 필드 여기에 누적)

## 이사님 보고용 정리 (최종 확정 시점에 작성)

> 마트 ETL 작업 완료 후 본 리스트를 최종 정리 → 이사님 보고

- 보고 내용 후보:
  - 필요 필드 표 (collector × 필드 × 우선순위)
  - 보강 후 영향: 마트 ETL 단순화 + H-DASH-01/03/18 방어 가능
  - 누가 작업: 사용자 직접
  - 일정: 마트 ETL 작업과 병행

## 관련 결함 패턴 (참조)

- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md#H-DASH-01]] — AppMode별 토큰 이중 카운트 (triggered_from 필요)
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md#H-DASH-03]] — 디버깅 데이터 혼입 (invoke_from, triggered_from 필요)
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md#H-DASH-18]] — Dify 예외 타입 DB 미보존 (workflow_runs.error도 audit에 있어야 ILIKE 분류 가능)

## 관련 노트

- [[3. 프로젝트/spx-agent/references/audit-schema.md]] — 현재 audit 스키마
- [[3. 프로젝트/spx-agent/references/dify-error-flow.md]] — 에러 흐름 분석 (collector → audit details 단계)
- [[1. Daily/2026-05-08.md]] — workflow-runs.ts 5필드 비대칭 첫 발견
- [[1. Daily/2026-05-11.md]] — 마트 설계 메모 § 3 (queryPreview 검토)
