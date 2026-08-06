---
tags: [프로젝트, dify, AI-Agent, HDD, infra]
type: spec/tasks
screen: 데이터 마트 (인프라)
harness: []
date: 2026-05-07
last_updated: 2026-05-15
---
# 데이터 마트 — Tasks

> **2026-05-15 진행 상태**: 0~2단계 (rename + collector P0 + Generated Column 8개 + 인덱스 5개) ✅ 적용 완료. **현재 위치 = 3단계 Layer 1 진입 직전**.
> 5/15 결정 4건 반영: Generated Column `_d` 접미사 / target_app_id UUID / is_debug에 rag-pipeline-debugging / collector `app.mode` LEFT JOIN 유지. 정식 명세는 [[3. 프로젝트/spx-agent/references/audit-details-spec.md]] 참조.

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/data-mart.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/data-mart.md|Design]]
> 상위 본문: [[3. 프로젝트/spx-agent/hdd/design.md]] § 2.5 마트 설계
> 결정 노트: [[0. Inbox/마트 설계 결정 - 2026-05-14.md]]

> 2026-05-14 구축 순서 7단계 체크리스트 (design § 2.5.7 = 결정 노트 § 10). 5/8 인벤토리/측정 단계는 결정 노트로 대체되어 본 spec에서 제외.

## 0단계: audit rename → public + `spx_` prefix ✅ 완료 (2026-05-15)

- [x] 0-1. `audit.audit_events` → `public.spx_audit_events` rename
- [x] 0-2. audit schema 부수 테이블 rename (collector_state/log_file_state/system_logs/system_log_batch → `spx_*`)
- [x] 0-3. trigger 함수 `audit.log_dify_change()` → `public.spx_log_dify_change()` + trigger 5종 재배포
- [x] 0-4. dify-audit collector INSERT target rename + Prisma schema rename
- [x] 0-5. `DROP SCHEMA audit CASCADE` + `setup_audit_schema.sql` CASCADE 보강 (사고 재발 방지)

## 1단계: collector P0 보강 ✅ 완료 (2026-05-15)

> 대상 repo: `dify-audit/src/lib/collectors/*.ts`. 명세: `references/audit-details-spec.md § P0 patch diff A.1~A.4`. **`app.mode` LEFT JOIN 유지** (5/15 결정 4 — messages.app_mode 직접 사용 안 함).

- [x] 1. `messages.ts` 보강 (+3줄)
  - [x] 1-1. SQL에 `app.mode AS app_mode` 추가 (기존 LEFT JOIN apps 재사용)
  - [x] 1-2. details 객체에 `appMode: r.app_mode` 추가
- [x] 2. `workflow-runs.ts` 보강 (+5줄)
  - [x] 2-1. SQL에 `app.mode AS app_mode`, `wr.triggered_from` 추가
  - [x] 2-2. details에 `appMode: r.app_mode`, `triggeredFrom: r.triggered_from` 추가
- [x] 3. `workflow-nodes.ts` 보강 (+3줄)
  - [x] 3-1. SQL에 `app.mode AS app_mode` 추가
  - [x] 3-2. details에 `appMode: r.app_mode` 추가
- [x] 4. `conversations.ts` 보강 (+3줄, `c.mode` 직접 사용)
  - [x] 4-1. SQL에 `c.mode AS app_mode` 추가 (NOT NULL 컬럼)
  - [x] 4-2. details에 `appMode: r.app_mode` 추가
- [x] 5. 통합 검증 — 4 collector 트리거 검증 시나리오 통과 (`references/audit-details-spec.md § 검증 시나리오`)

> 총 변경 ~14줄, JOIN 추가 0건.

## 2단계: spx_audit_events Generated Column 8개 + 인덱스 5개 ✅ 완료 (2026-05-15)

> 도구: **Prisma Migrate (raw SQL migration 패턴)** — Alembic 아님. 폴더: `dify-audit/prisma/audit/migrations/`.

- [x] 6. `pnpm prisma migrate dev --create-only --name add_generated_columns_for_mart` → 빈 migration 생성
- [x] 7. PG 버전 확인 → **15.15** (STORED Generated Column PG 12+ 요구 충족)
- [x] 8. Generated Column 8개 ALTER (`_d` 접미사 컨벤션, design § 2.1)
  - [x] 8-1. `app_mode_d`, `model_provider_d`, `model_id_d`, `total_tokens_d`(BIGINT), `error_d`, `invoke_from_d`, `triggered_from_d`, `target_app_id`(UUID)
- [x] 9. 인덱스 5개 추가 (마트 가속용)
  - [x] 9-1. `spx_audit_events_action_mart_idx` (partial, action IN ('message_send','workflow_execute'))
  - [x] 9-2. `spx_audit_events_tenant_target_app_idx` (tenant_id, target_app_id)
  - [x] 9-3. `spx_audit_events_tenant_actor_idx` (tenant_id, actor_id)
  - [x] 9-4. `spx_audit_events_occurred_action_idx` (occurred_at, action)
  - [x] 9-5. `spx_audit_events_app_mode_idx` (app_mode_d)
- [x] 10. downgrade 작성 (DROP COLUMN 역순)
- [x] 11. 마이그레이션 적용 — `prisma migrate resolve --applied` (audit_writer ALTER 권한 없어 postgres superuser로 직접 DDL 실행 후 정합 처리)

> **운영 적용 시 주의 (5/15 사고 학습)**: `CREATE INDEX CONCURRENTLY`는 트랜잭션 밖이라 migration에서 사용 불가 → 별도 SQL로 수동 실행 권장. `audit_writer` role ALTER 권한 부재로 `prisma migrate deploy` 실패 가능 → superuser 직접 DDL or 사전 GRANT.

## 3단계: Layer 1 — spx_mv_audit_enriched MView + spx_v_resource_ownership_enriched View ← **현재 위치 (다음 작업)**

> ⚠️ `mart` schema 생성 단계 폐기 — 5/14 결정으로 모든 마트 객체는 `public` schema에 `spx_` prefix.

- [ ] 13. `public.spx_mv_audit_enriched` MView 작성 (design § 2.2)
  - [ ] 13-1. `_d` 컬럼을 마트 친화 alias로 노출 (`ae.app_mode_d AS app_mode` 등)
  - [ ] 13-2. `spx_resource_ownership` INNER JOIN (결정 a — 매칭 안 되면 사라짐, 모니터링 #3가 캐치)
  - [ ] 13-3. `spx_department_members` LEFT JOIN (end_user/api 보존)
  - [ ] 13-4. `is_debug` = `invoke_from_d = 'debugger' OR triggered_from_d IN ('debugging','rag-pipeline-debugging')` (5/15 결정)
  - [ ] 13-5. `is_canonical_call` = `app_mode_d <> 'advanced-chat' OR action = 'message_send'`
  - [ ] 13-6. `WHERE action IN ('message_send','workflow_execute')`
  - [ ] 13-7. UNIQUE INDEX (id) — CONCURRENTLY 전제
- [ ] 14. `public.spx_v_resource_ownership_enriched` View 작성 (design § 2.5)
  - [ ] 14-1. INNER JOIN spx_departments (NULL은 모니터링 #4가 캐치)
- [ ] 15. 검증
  - [ ] 15-1. enriched 행수 vs spx_audit_events 대상 action 행수 비교 (메트릭 #3 의미)
  - [ ] 15-2. `is_debug=TRUE` 비율 (디버깅 호출 비율 확인) — `debugging` + `rag-pipeline-debugging` 양쪽 포함됐는지
  - [ ] 15-3. `is_canonical_call=FALSE`인 행이 advanced-chat의 workflow_execute뿐인지 확인

## 4단계: Layer 2 — spx_mv_kpi_calls_daily + spx_mv_model_tokens_daily

- [ ] 16. `public.spx_mv_kpi_calls_daily` MView 작성 (design § 2.3)
  - [ ] 16-1. `WHERE NOT is_debug AND is_canonical_call` 필터
  - [ ] 16-2. GROUP BY (day, app_owner_dept_id, actor_dept_id, target_app_id, app_mode)
  - [ ] 16-3. UNIQUE INDEX (tenant_id, day, app_owner_dept_id, actor_dept_id, target_app_id, app_mode)
- [ ] 17. `public.spx_mv_model_tokens_daily` MView 작성 (design § 2.4)
  - [ ] 17-1. `WHERE NOT is_debug AND action='message_send' AND total_tokens IS NOT NULL`
  - [ ] 17-2. UNIQUE INDEX (tenant_id, day, model_provider, model_id)
- [ ] 18. KST day 경계 확인 — `date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul')`
- [ ] 19. 카디널리티 확인 — 운영 부서 N + day 30 + 앱 × app_mode 기준 백만 행 이내

## 5단계: Collector trigger 연동 chain refresh + 모니터링 6개

> pg_cron 미채택 (2026-05-15 결정). dify-audit db-poller가 polling cycle 끝에 직접 호출. design § 4 참조

- [ ] 20. dify-audit 컨테이너 안의 db-poller 파일 위치 확인 (`src/workers/db-poller.ts` 또는 비슷한 파일)
- [ ] 21. `refreshMartChain()` 함수 추가 (design § 4.1 윤곽)
  - [ ] 21-1. enriched → daily_calls → daily_model_tokens 순차 호출 (`prisma.$executeRawUnsafe`)
  - [ ] 21-2. 모두 `REFRESH MATERIALIZED VIEW CONCURRENTLY` (UNIQUE INDEX 전제)
  - [ ] 21-3. try/catch + logger — 마트 실패가 collector polling 다음 회차 차단하면 안 됨
- [ ] 22. db-poller run 끝에 `await refreshMartChain()` 호출 추가
- [ ] 23. dify-audit 컨테이너 재빌드 + 재시작 (`docker compose build dify-audit && up -d`)
- [ ] 24. 모니터링 메트릭 6개 쿼리 작성 (design § 5)
  - [ ] 24-1. #1a collector lag / #1b mart lag / #2 chain 소요 (collector 로그 시각 차) / #3 행수 차이 / #4 dept NULL / #5 actor_dept NULL 비율
  - [ ] 24-2. 임계 위반 시 알람 경로 (Slack `#aai-alerts` 검토)
- [ ] 25. 검증
  - [ ] 25-1. 컨테이너 재시작 직후 collector 로그에 `[mart] chain refresh completed` 박힘 확인
  - [ ] 25-2. 5분 cron 1회 추가 후 3 mat 모두 신규 행 채워짐 확인
  - [ ] 25-3. chain refresh 소요 < 4분 (`[db-poller] Run completed` ~ `[mart] chain refresh completed` 시각 차)
  - [ ] 25-4. 마트 실패 주입 테스트 (예: enriched DROP) — collector polling 다음 회차 정상 진행 확인 (try/catch 동작)

## 6단계: 백엔드 service 재작성 — Layer 2 enforce (11종)

> 컴포넌트 endpoint 5종 + drill-through 11종이 Layer 2만 보도록 enforce. 컴포넌트가 raw `details->>` 추출 사용 시 H-DASH-01/03 방어 누락.

- [ ] 24. SQLAlchemy 모델 클래스 (`api/models/mart.py`, 신규 파일)
  - [ ] 24-1. `VAuditEnriched`, `VResourceOwnershipEnriched`, `MvKpiCallsDaily`, `MvModelTokensDaily` 4종 매핑
  - [ ] 24-2. **Alembic autogen 제외** — 각 클래스 `__table_args__ = {'info': {'is_view': True}}` 마커. alembic env 설정도 View/MView 무시 확인
  - [ ] 24-3. **권한 확인** — 4종 모두 `audit_writer` 소유 + `GRANT SELECT` 적용됨 (5/15 Layer 2 작업 시 박힘). 미적용 시 superuser 직접 `ALTER OWNER TO audit_writer`
  - [ ] 24-4. **컬럼 시그니처 검증** — `design.md § 2.5.4` DDL과 1:1 대조. PowerShell + docker exec에선 `psql \d+` escape 안 먹힘 → `SELECT column_name, data_type FROM information_schema.columns WHERE table_name='spx_mv_audit_enriched'` 우회 (5/15 학습)
- [ ] 25. kpi-cards 서비스 재작성 (`dashboard_kpi_service.py`)
  - [ ] 25-1. #1 총 오브젝트 → `spx_v_resource_ownership_enriched`
  - [ ] 25-2. #2 채택 폭 (actor 기준) → `spx_mv_kpi_calls_daily` + GROUP BY actor_dept_id
  - [ ] 25-3. #3 API 호출 (owner 기준) → `spx_mv_kpi_calls_daily` + GROUP BY app_owner_dept_id
  - [ ] 25-4. #4 Top 앱 → `spx_mv_kpi_calls_daily` + apps(name) JOIN
- [ ] 26. dept-objects 서비스 → `spx_v_resource_ownership_enriched`
- [ ] 27. dept-activity 서비스 (6컬럼) → `spx_mv_kpi_calls_daily` (호출/토큰) + `spx_v_resource_ownership_enriched` (신규 a/k/t)
- [ ] 28. model-tokens 서비스 → `spx_mv_model_tokens_daily`
- [ ] 29. drill-through 11종
  - [ ] 29-1. 카드/표 헤드 = `spx_mv_kpi_calls_daily` 합산 (근사)
  - [ ] 29-2. drill 표만 enriched 직조회 `COUNT DISTINCT` (정확 users — 미해결 결정 d)
- [ ] 30. grep 검증 (**PR 직전 필수 게이트** — 1건이라도 위반 시 PR 차단. 결과는 실제 명령 출력 그대로 보고할 것 — 5/15 자가 검증 거짓 학습)
  - [ ] 30-1. `grep -rE "details->>'(invokeFrom|triggeredFrom|appMode|modelProvider|modelId|totalTokens|errorText)'" api/services/admin/` → **0 hit** (컴포넌트가 raw 추출하면 H-DASH-01/03 방어 누락)
  - [ ] 30-2. `grep -rE "(app_mode_d|triggered_from_d|invoke_from_d|model_provider_d|model_id_d|total_tokens_d|error_d)" api/services/admin/` → **0 hit** (마트 인터페이스 우회, enriched alias만 사용)
  - [ ] 30-3. `grep -rE "FROM\s+(messages|workflow_runs|workflow_node_executions)\b" api/services/admin/` → **0 hit** (raw 테이블 직접 SELECT, AppMode 분기 / 디버깅 필터 누락 위험)
  - [ ] 30-4. `grep -rE "(spx_mv_audit_enriched|spx_mv_kpi_calls_daily|spx_mv_model_tokens_daily|spx_v_resource_ownership_enriched)" api/services/admin/` → **hit 있음** (Layer 2 / enriched view 사용 확인)

## 7단계: 부하 테스트 — p95 < 3초 임계 검증

- [ ] 31. mock 데이터 환경 셋업 (운영 카디널리티 가정 — 부서 수십, 앱 수십, 일 30, app_mode 3 → 백만 행 단위)
- [ ] 32. 16종 endpoint (5 컴포넌트 + 11 drill) 응답 시간 측정
  - [ ] 32-1. p50 / p95 기록
  - [ ] 32-2. 임계 위반 endpoint 식별
- [ ] 33. 위반 시 진단
  - [ ] 33-1. EXPLAIN ANALYZE — 인덱스 미사용 의심 시 인덱스 추가
  - [ ] 33-2. 카디널리티 분석 — 부서 N 크면 부분 인덱스 검토
  - [ ] 33-3. Read Replica 검토 (미해결 결정 e)
- [ ] 34. 결과 SESSION_HISTORY.md 기록

## 검증 체크리스트 (단계별 게이트)

| 단계 | 통과 조건 |
|------|---------|
| 1단계 collector | 4 collector 트리거 후 spx_audit_events에 새 키 채워짐 확인 |
| 2단계 Generated Column | enriched에서 컬럼 NULL이 정상 NULL뿐인지 (workflow_execute의 model_* 등) |
| 3단계 Layer 1 | 메트릭 #3 (enriched 행수 차이) = 0 |
| 4단계 Layer 2 | daily mat 행수 vs enriched 직집계 일치 |
| 5단계 cron + 모니터링 | mart_lag < 7분 + chain 소요 < 4분 |
| 6단계 service | 모든 endpoint가 Layer 2/ownership_enriched로만 조회, raw `details->>` 사용 0 |
| 7단계 부하 | 16종 endpoint p95 < 3초 |

## 참조

- requirements: [[3. 프로젝트/spx-agent/hdd/specs/requirements/data-mart.md]]
- design: [[3. 프로젝트/spx-agent/hdd/specs/design/data-mart.md]]
- 상위 본문: [[3. 프로젝트/spx-agent/hdd/design.md]] § 2.5
- 5/14 결정 노트: [[0. Inbox/마트 설계 결정 - 2026-05-14.md]]
- audit 필드 명세 + P0 collector 보강 명세: `references/audit-details-spec.md`
- 회의록: [[2. 회의록/0507 AAI 주간 보고]]
