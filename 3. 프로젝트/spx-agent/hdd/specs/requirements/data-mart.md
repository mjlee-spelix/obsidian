---
tags: [프로젝트, dify, AI-Agent, HDD, infra]
type: spec/requirements
screen: 데이터 마트 (인프라)
harness: []
date: 2026-05-07
last_updated: 2026-05-26
---
# 데이터 마트 — Requirements

> **2026-05-15 진입 차단 결정 4건 ✅ 확정 + 적용 완료**:
> 1. Generated Column `_d` 접미사 컨벤션 (8개 중 7개, target_app_id 예외)
> 2. `target_app_id` UUID 타입 (resource_ownership.resource_id 직접 JOIN)
> 3. `is_debug` 표현식에 `rag-pipeline-debugging` 추가 (`triggered_from_d IN ('debugging','rag-pipeline-debugging')`)
> 4. collector 4건 `app.mode` LEFT JOIN 유지 (messages.app_mode 직접 사용 안 함)
>
> rename (audit → public + spx_) + collector P0 4건 + Generated Column 8개 + 인덱스 5개 모두 2026-05-15 적용 완료. **현재 위치 = Layer 1 진입 직전**.

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/data-mart.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/data-mart.md|Tasks]]
> 회의 결정: [[2. 회의록/0507 AAI 주간 보고]]
> 상위 본문: [[3. 프로젝트/spx-agent/hdd/design.md]] § 2.5 마트 설계
> 사람용 컨텍스트: [[0. Inbox/마트 설계 결정 - 2026-05-14.md]]

> ⚠️ 본 spec은 **화면 컴포넌트가 아닌 인프라 레이어**.
> `mount_environment` / `design_image` 등 화면 spec frontmatter 항목은 N/A.

## 배경

대시보드 5개 컴포넌트(kpi-cards / kpi-drill-through / dept-objects / dept-activity / model-tokens) + drill-through 11종이 OLTP를 직접 조회하면 응답 느림 + Harness 방어(H-DASH-01/03)가 컴포넌트마다 흩어짐. **spx_audit_events 단일 SoT + RBAC JOIN**으로 마트화하여 응답 속도 + 방어 일관성 동시 확보 (2026-05-12 이사님 확정).

## 마트 입력 요구사항 (2026-05-14 확정)

### 단일 SoT + 분기 3건

| # | 항목 | 결정 |
|---|---|---|
| - | 입력 소스 | **spx_audit_events 단일** + RBAC JOIN. 오브젝트 차트만 `resource_ownership` 직접 (state 본질) |
| a | "신규 X" 정의 | `resource_ownership.created_at` 단일 기준 (apps.created_at 분기 폐기). 레거시 앱 없음 → `INNER JOIN ro`, "미배정" 버킷 폐기 |
| b | "호출 수" 부서 기준 | **owner 기준** 일관 (KPI #3, dept-activity 호출, drill-through 모두) |
| c | api_call(nginx) | **제외**, `message_send` + `workflow_execute` 두 액션만 |

> 근거: [[0. Inbox/부서별 API 호출 집계 - api_call 대신 messages+workflow_runs 쓰는 이유.md]]

### RBAC JOIN 전략 — 옵션 B (사전 JOIN하되 ID만)

- enriched MView에 `app_owner_dept_id`, `actor_dept_id` (ID만) 보존
- 부서명(`departments.name`)은 컴포넌트/Layer 2 쿼리에서 query-time JOIN
- 부서명 변경 즉시 반영, ownership·membership 변동은 refresh 주기(5분) stale
- 비싼 결합(audit × dept_members 만 단위 × ownership)은 refresh 시 1회로 흡수

### 채택 vs 호출 부서 기준 분리

| 컴포넌트 | 의미 | 부서 기준 |
|---|---|---|
| kpi-cards #2 총 이용 앱 수 (Breadth) | 부서원이 실제로 이용하는 앱 종류 수 | **actor** |
| kpi-cards #3 총 앱 호출량 (Depth) | 부서가 소유한 앱이 받은 호출량 | **owner** |
| kpi-cards #4 인기 호출 앱 / drill-through | 호출량 기반 Top 앱 | **owner** |
| dept-activity 호출/토큰 컬럼 | 부서 소유 앱의 활동량 | **owner** |
| dept-activity 신규 앱/지식/도구 컬럼 | 부서가 새로 보유한 자산 | **owner** (ownership.owner_dept) |

→ Layer 2 mat에 `actor_dept_id` + `app_owner_dept_id` **둘 다 GROUP BY 컬럼**으로 보존.

## 응답 시간 목표

| 지표 | 목표 | 비고 |
|------|------|------|
| 단건 KPI 응답 (p95) | < 1s | |
| 차트 1종 응답 (p95) | < 2s | |
| 페이지 전체 첫 페인트 | < 3s | 회의 결정 (5/7) |
| 마트 신선도 SLA | < 7분 (mart lag) | chain refresh 5분 + buffer 2분 |
| collector lag | < 10분 | 폴링 5분 + buffer 5분 |

## 영향받는 컴포넌트 — 마트 객체 의존 매핑

| 컴포넌트 | 사용 객체 | 기준 |
|---|---|---|
| kpi-cards #1 총 오브젝트 | spx_v_resource_ownership_enriched | state |
| kpi-cards #2 총 이용 앱 수 | spx_mv_kpi_calls_daily | actor |
| kpi-cards #3 총 앱 호출량 | spx_mv_kpi_calls_daily | owner |
| kpi-cards #4 인기 호출 앱 | spx_mv_kpi_calls_daily + apps(name) | owner |
| kpi-drill-through 4구역 | spx_mv_kpi_calls_daily + apps(name) + 정확 users는 spx_mv_audit_enriched | owner |
| dept-objects | spx_v_resource_ownership_enriched | state |
| dept-activity 6컬럼 | spx_mv_kpi_calls_daily + spx_v_resource_ownership_enriched | 호출/토큰=owner, 신규=ownership.created_at |
| model-tokens | spx_mv_model_tokens_daily | — |

## 지원해야 하는 분석 차원

- **시간**: day (KST 경계). 필요 시 hour 별도 mat
- **부서**: `app_owner_dept_id` + `actor_dept_id` (둘 다 ID로 보존, 부서명 query-time)
- **앱/모델**: `target_app_id` (UUID, 직접 JOIN), `app_mode_d`, `model_provider_d`, `model_id_d` (enriched에서 `app_mode`/`model_provider`/`model_id` alias)
- **사용자**: `actor_id` COUNT DISTINCT (drill 정확값은 enriched 직조회)
- **리소스 유형**: `resource_type` (App / KB / Tool) — ownership_enriched

## 지원해야 하는 측정값

- 호출 횟수 (`calls`)
- 토큰 합계 (`tokens` — enriched에서 `total_tokens_d` BIGINT를 `total_tokens`로 alias)
- 에러 건수 (`errors` — `error_d IS NOT NULL`)
- 활성 사용자 수 (`users` — daily 합산은 근사, 정확값은 enriched COUNT DISTINCT)
- 마지막 발생 시각 (`last_seen`)

## 보존 기간

- spx_audit_events 자체: 90일 (2026-05-12 이사님 확정)
- daily mat: audit 보존 따름 (audit 만료 후 daily mat은 별도 archive 검토 — Phase B)

## 데이터 신선도

- **refresh = Collector trigger 연동** (dify-audit db-poller worker가 5분 polling cycle 끝에 chain refresh 호출. pg_cron 미채택 — 2026-05-15 결정. design § 4 참조)
- 사용자 체감 lag 최악 10분 (collector 5분 + chain 5분), TanStack staleTime 5분과 정합
- 실시간성 요구사항 없음 (배치 우선 — 진화 경로는 design § 2.5.6)

## Harness 방어 — 마트 인터페이스에 흡수

> 컴포넌트가 SQL 짤 때마다 누락 가능한 방어를 마트 인터페이스에 강제로 흡수.

| 규칙 | 출처 | 마트 반영 |
|------|------|----------------|
| AppMode 분기 (ADVANCED_CHAT은 message_send만) | H-DASH-01 | enriched의 `is_canonical_call` 플래그 컬럼 (마트 인터페이스 강제) |
| 디버깅 데이터 제외 | H-DASH-03 | enriched의 `is_debug` 플래그 컬럼 + daily mat 필터 `WHERE NOT is_debug` |
| 미배정 처리 | H-DASH-04 | `INNER JOIN ro` (결정 a, 매칭 안 되면 모니터링 #3 캐치) |
| 이전 기간 데이터 부재 | H-DASH-05 | 마트 자체엔 N/A 컬럼 X, 계산은 서비스 레이어에서 |
| 워크플로 모델 정보 누락 | H-DASH-02 | spx_mv_model_tokens_daily는 `action='message_send'`만 |

## 외부 의존 (블로커)

| 의존 | 상태 | 비고 |
|------|------|------|
| **collector P0 보강** | ✅ 완료 (2026-05-15) | 4 collector × `appMode` + workflow-runs `triggeredFrom`. 패치 4건 적용 (`app.mode` LEFT JOIN 유지). 명세: `references/audit-details-spec.md § P0 patch diff A.1~A.4` |
| RBAC 테이블 DDL | ✅ 확정 (2026-05-06) | 권대리님 `feat/rbac` + 우리 docker DB 영속 |
| Generated Column STORED 지원 PG 버전 | ✅ 확정 (PG 15.15) | PG 12+ 요구사항 충족. 5/15 마이그레이션 완료 |

## 비기능 요구사항

- **CONCURRENTLY 가능**: 모든 MView에 UNIQUE INDEX 필수 (refresh 중 SELECT 가능)
- **모니터링 메트릭 6개**: collector lag / mart lag / chain refresh 소요 / 행수 차이 / dept NULL / actor_dept NULL 비율 (design § 2.5.5)
- **백필 시점**: 2026-05-15 마이그레이션 시점에 데이터 작은 수준이라 Generated Column ALTER 백필 lock 비용 사실상 0. 운영 진입 시점부터 자연 충족
- **Layer 2가 컴포넌트 인터페이스**: incremental fact / 분석DB 전환 시에도 컴포넌트 코드 무수정

## 명명 규칙 (2026-05-14 — public schema + `spx_` prefix)

- **모든 객체 `public` schema** (audit/mart/rbac 별도 schema 폐기 — Dify upstream + 회사 RBAC와 통일)
- **우리 객체에 `spx_` prefix** — Dify upstream(`apps`, `messages` 등 — 무접두)과 시각적 구분. 회사 RBAC 표준 5종(`spx_departments` 등)도 2026-05-19 `spx_` 접두사로 통일되어 우리 객체와 prefix 공유
- 객체:
  - audit: `spx_audit_events`
  - Layer 1: `spx_mv_audit_enriched` (MView), `spx_v_resource_ownership_enriched` (View)
  - Layer 2: `spx_mv_kpi_calls_daily`, `spx_mv_model_tokens_daily`
- 마트 refresh 트리거: collector worker (db-poller) — pg_cron 미채택 (2026-05-15 결정)
- 마트 메타: collector 로그 + 운영 진입 시 `spx_mart_refresh_log` 테이블 도입 검토

## 미해결 결정 — design.md § 10 참조

drill-through users 정확도 / Read Replica 분리 / 타임존 / dbt 도입 / PM 확인 2건은 design.md § 10 미해결 결정 표 참조.

## 참조

- 상위 본문: [[3. 프로젝트/spx-agent/hdd/design.md]] § 2.5
- 회의록: [[2. 회의록/0507 AAI 주간 보고]]
- 5/14 마트 설계 결정 노트: [[0. Inbox/마트 설계 결정 - 2026-05-14.md]]
- 기존 OLTP 스키마: `references/dify-db-schema.md`
- audit 필드 명세: `references/audit-details-spec.md`
- RBAC 스키마: `references/rbac-schema.md`
- TanStack staleTime 관계: 클라이언트 캐시(staleTime 5분)와 별개 레이어. 둘 다 유지.
