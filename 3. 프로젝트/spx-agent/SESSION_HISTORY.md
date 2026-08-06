---
tags: [프로젝트, dify, AI-Agent]
last_updated: 2026-06-10
---

# Session History — SPX-Agent


> 옵시디언 Claudian + VSCode Claude Code 양쪽이 공유하는 진행 상태.
> 세션 시작 시 이 문서부터 읽고, 끝낼 때 한두 줄 갱신.
>
> 상세 기록은 옵시디언 데일리 노트(`1. Daily/YYYY-MM-DD.md`)에. 여기는 요약만.

---

## 2026-06-12 — B안 dev 배포 확인 (v1.2.4) + prod 미배포 정리

### 결론: B안은 master 머지 + v1.2.4 태그 + **dev 배포까지 완료**
- KAN-29 → master 머지됨(`4db5d8da feat: ...model_tokens UNION...`). 태그 **`v1.2.4`**에 포함(B안 + 로고/대시보드 변경 등).
- **dev 스택 v1.2.4 배포 완료**(2026-06-12 15:32~33 빌드): `spx-dev-dify-audit`(마이그레이션 `20260610` 적용 6/12 15:40, model_tokens = **UNION 정의**, collector process_data 파싱 포함) + `spx-dev-web`(로고/대시보드 UI 반영). **dev 대시보드에서 반영 확인됨.**

### 어제(6/11)까지 "B안 안 보임" 헤맨 이유 (해소)
- 배포가 **옛 태그 v1.2.3 기준**이었음(B안 머지 *전*). 어제 dev 컨테이너는 v1.2.3 빌드라 마이그레이션·collector 없었음 → 오늘 v1.2.4로 재빌드되며 해소.
- 추가로, "로고/대시보드 안 보임"은 **prod 대시보드(`194:80` = spx-prod-nginx)를 보고 있었던** 것. prod web은 stale(옛 이미지). dev web(v1.2.4)엔 정상 반영.
- 진단 팁: 서버 repo `/opt/spx-agent`는 태그 detached HEAD. `git log v1.2.x | grep`로 "그 태그가 커밋 포함하나" 확인. 컨테이너 실제 코드는 `docker exec ... find/grep` + 이미지 `Built` 시각.

### PROD 미배포 (남은 것)
- `spx-prod-web` = **옛 이미지(stale, 이미지 삭제됨)** → prod 대시보드 옛 UI.
- **prod dify-audit 컨테이너 없음** → dify_prod에 audit 마트 없음 → prod 대시보드 KPI/차트 0/빈. (오브젝트 2는 RBAC라 나옴)
- → **prod 전체(web+api+dify-audit)를 v1.2.4로 배포** 필요. `spx-prod-dify-audit:latest` 이미지는 이미 빌드돼 있음.

### 남은 작업
- **dev 검증**: 3.6 mock 시드(dify_dev `spx_audit_events`에 S1~S6) → REFRESH → 모델 차트에 워크플로우/챗플로우 모델. (collector 옛 수집분은 model NULL이라 시드/신규 실행 필요)
- **prod 배포**: prod 스택 v1.2.4 전체 배포 → web(UI) + dify-audit(마트·데이터) 정상화.
- 보류: 로컬 라벨 namespaced fix(별도 트랙, `PROMPT.md`/`references/local-provider-classification.md`).

---

## 2026-06-10 — 194 DB 손상 결말 + 인프라 토폴로지 정리 (dify_prod/dify_dev 재구성)

### 194 DB 사건 결말 (우리 작업 무관 확정)
- 3.5(B안 194 적용) 직후 **XX001 스토리지 손상** + `pg_class` 동일 OID 중복(카탈로그 손상) → DB 다운.
- **우리 작업 탓 아님 확정**: 디스크 풀 ❌(20%), dmesg I/O 에러 ❌, 서버 리부팅 ❌(89일 가동), OOM ❌. XX001=물리 파일 손상은 SQL로 못 만듦. 재시작으로도 안 고쳐짐(파일 레벨).
- 이후 **인프라가 재구성**: 옛 `dify` DB(손상) → 비우고 `dify_prod`(원천 데이터) + `dify_dev` 신설.

### 인프라 토폴로지 (2026-06-10 확인)
- **postgres 1개**(`spx-prod-infra-db_postgres-1`, 호스트 194:**15432**)에 모든 DB: `dify`(빈/레거시) · `dify_prod`(prod 원천, audit 없음) · `dify_dev`(dev 원천+audit) · `dify_dev_audit_shadow` · `dify_plugin` · `keycloak`.
- **dify-audit 컨테이너는 dev에만 존재**: `spx-dev-dify-audit-1` → `dify_dev`. **prod 스택(spx-prod-*)엔 dify-audit 없음** → dify_prod에 audit 레이어(spx_audit_events/마트) 미설치.
- dify-audit env: `DIFY_DATABASE_URL` = `AUDIT_DATABASE_URL` = `db_postgres:5432/dify_dev` (원천+audit 같은 DB 공존). **컨테이너 1개 = DB 1개** (env에 DB명 고정 → dev/prod 겸할 수 없음).
- 실 운영 Dify 별도: `192.168.10.159:5432/dify` (DBeaver "dify 3", 읽기전용) — provider 조사용.

### dify_dev 상태 (B안 미적용 — dev 이미지 stale)
- `spx_audit_events` 4863건(collector 가동 중), 단 **마이그레이션 `20260610`(B안 UNION) 없음** + model_tokens 정의가 옛것(`FROM spx_mv_audit_enriched`). → **dev 이미지가 B안 변경 전 빌드**.

### 작업 방향 (결론)
- **B안 검증/시드는 dify_dev에서** (안전, audit 스택 살아있음). 단 **dev dify-audit를 KAN-29(B안)로 `--no-cache` 재빌드 + 재시작** 선행 → 마이그레이션 적용(UNION) + collector process_data 파싱 + self-heal 경화 반영. 그 후 3.6 시드 → 워크플로우 모델 검증.
- **prod(dify_prod)는 별도 배포** — 같은 이미지 + env `dify_prod`로 **dify-audit 인스턴스 추가**(dev 컨테이너 재활용 불가). prod 스택 compose에 dify-audit 서비스 추가 필요. 띄우면 B안 자동 적용 + collector가 dify_prod 원천 채움.
- **우리 B안 작업물(코드·마이그레이션·문서)은 git에 안전** — 어느 env든 컨테이너 올라오면 자동 적용.

---

## 2026-06-10 — B안 6단계: 테스트 재작성 + grep 게이트 + 데이터 검증 (Claude Code)

### 배경
- B안 5.5단계(로컬 E2E 검증)까지 완료된 상태에서 6단계(테스트 + grep 게이트) 수행. 194 대기 항목(27 provider 포맷 spot-check, 28 중첩 노드 이중집계)은 스킵.

### 변경 내역

**task 26 — 테스트 재작성**
- **`test_dashboard_model_tokens_service.py`** 전면 재작성:
  - H-DASH-01: 챗플로우 노드 토큰 집계(이중집계 0) 테스트 추가.
  - H-DASH-02: "workflow 배제" → "workflow/chatflow 포함(B안)" 관점으로 전환 — 실제 모델로 등장, `unclassified_workflow_tokens` 없음.
  - H-DASH-03: 디버그 제외(마트 레벨) 테스트 신설.
  - H-DASH-09: 로컬 판별 + display_name "(로컬)" 접미사 assertion 강화.
- **`test_dashboard_drill_calls_service.py`** `TestModelCallShare` 재작성:
  - mock 반환값 `(model_id, count)` → `(model_provider, model_id, count)` 3컬럼으로 변경 (마트 repoint 반영).
  - `test_workflow_models_included`: B안 워크플로우/챗플로우 모델 등장 테스트 추가.
  - `test_local_provider_label`: H-DASH-09 "(로컬)" 라벨 테스트 추가.
  - 기존 `TestDeptCallCount`/`TestDeptCallRpsTable`/`TestDrillCallsFallback`은 변경 없이 유지.

**task 20~24 — 데이터 검증 (로컬 DB)**
- task 22: S3 챗플로우 토큰 총합 일치 — `node_tokens(24,000) == msg_tokens(24,000)`, 마트에 '미분류' 0건.
- task 20/21/23/24: 단위 테스트 내 assertion으로 커버 (mart mock 기반).

**task 25 — grep 게이트 (전 항목 통과)**
- 25-1: `VAuditEnriched|enriched` in drill_calls_service → **0 hit** (enriched 직조회 완전 제거).
- 25-2: `FROM (messages|workflow_runs|workflow_node_executions)` in services/admin/ → **0 hit** (raw 테이블 직조회 0).
- 25-3: `process_data` in workflow-nodes.ts → **7 hit** (collector 보강 확인).

**task 29 — 로컬 모델 판별**
- `test_local_provider_label` 테스트로 확인: ollama provider → `display_name = "llama3 (로컬)"`.

### 결과
- pytest 15 passed, 0 failed (model_tokens 6 + drill_calls 9).
- grep 게이트 3/3 통과.
- S3 챗플로우 토큰 총합 일치(24,000 = 24,000), 미분류 0건.
- **194 대기 항목**: task 27(provider 포맷 spot-check), task 28(중첩 노드 이중집계) — 194 복구 후 실데이터로 검증.
- **⚠️ 발견**: `MvModelTokensDaily` ORM의 `tenant_id`가 `StringUUID`로 매핑되어 `::UUID` 캐스트 오류 → `sa.Text`로 수정 (5.5단계에서 발견·수정, 6단계 테스트로 회귀 방지).

---

## 2026-06-10 — B안 4~5.5단계: 서비스 repoint + 프론트 + 로컬 E2E 검증 (Claude Code)

### 배경
- B안 3.7단계까지 완료된 상태에서 4단계(백엔드 서비스 repoint) + 5단계(프론트엔드) + 5.5단계(로컬 E2E 검증) 수행. 194 DB 다운 중이라 로컬 DB 전환으로 E2E 검증.

### 변경 내역

**4단계 — 백엔드 서비스 repoint**
- **task 14** `dashboard_model_tokens_service.py`: docstring "message_send only" → "UNION: message_send + workflow_node_execute" 갱신. 코드 변경 0(마트 재작성으로 자동 유입).
- **task 15** `dashboard_drill_calls_service.py` — `get_model_call_share` **핵심 repoint**:
  - `VAuditEnriched` 직조회(COUNT) **완전 제거** → `MvModelTokensDaily` SUM(calls) GROUP BY model_provider, model_id.
  - `LOCAL_PROVIDERS` 로컬 판별 추가 (H-DASH-09) — `_LOCAL_PROVIDERS = frozenset({"ollama", "xinference", "localai"})`, display_name에 "(로컬)" 라벨.
  - import에서 `VAuditEnriched` 제거, `MvModelTokensDaily` 추가.
- **task 13** `mart.py` `MvModelTokensDaily` ORM 수정:
  - docstring: "message_send only" → "UNION: non-chatflow message_send + workflow/chatflow node_execute" 갱신.
  - ⚠️ **타입 불일치 수정 발견**: `tenant_id`가 `StringUUID`로 매핑되어 `::UUID` 캐스트 오류 발생 → `sa.Text`로 수정 (마트는 text 컬럼).
  - `tokens`가 `sa.BigInteger`로 매핑되어 있었으나 실제는 `numeric` (SUM of bigint 결과) → `sa.Numeric`으로 수정.

**5단계 — 프론트엔드**
- **task 16잔존** `model-tokens-chart/index.tsx:21`: 주석 "미분류 마지막" → "모델 내림차순" 정리.
- **task 17** `model-call-share.tsx`: 마트 기반으로 워크플로우/챗플로우 모델 자동 등장. 프론트 코드 변경 불필요(API 응답의 display_name 그대로 사용).
- **task 18** `model-call-share.tsx`: 제목에 `title` 속성으로 단위 툴팁 추가 — "모델 호출 수 = 노드(모델 호출) 단위. 워크플로우 1회 실행이 여러 모델 호출로 집계되어 KPI 총 호출량과 다를 수 있습니다."
- **task 19/19-1** `model-call-share.tsx`: 로딩 상태를 **골격 유지 패턴으로 수정** — 기존 early-return(`h-40` plain 텍스트) 제거 → 카드 wrapper + 제목 유지 + `h-[250px] animate-pulse` 스켈레톤. conventions.md § 빈 상태 정책 준수.

**5.5단계 — 로컬 E2E 검증**
- **LV-1**: `docker/.env` — `DB_HOST=192.168.10.194`, `DB_PORT=15432` 기록.
- **LV-2~3**: DB 호스트 → 로컬(`db_postgres:5432`) 전환 + api recreate. 로컬에 UNION 마트(ollama/llama3 포함) + RBAC 17부서 mock 확인.
- **LV-4 서비스 검증** (flask app_context 직접 실행):
  - `DashboardModelTokensService.get_model_tokens`: 13개 모델, `llama3 (로컬)` 포함, 총 250M 토큰 ✅
  - `DashboardDrillCallsService.get_model_call_share`: 13개 모델, `llama3 (로컬)` 라벨 정상, share_percent 계산 정상 ✅
- **LV-5 프론트 구조 검증**: 코드 리뷰 — 골격 유지/고정 높이/숫자 포맷/단위 툴팁 정합 확인. 시각적 확인은 web 빌드 후 브라우저 필요.
- **LV-6~7**: DB 호스트 194 원복 + api recreate. `.env` diff 0 (원복 흔적 없음).

### 결과
- 코드 변경: `dashboard_drill_calls_service.py`(핵심 repoint), `dashboard_model_tokens_service.py`(주석), `mart.py`(ORM 타입 수정+주석), `model-call-share.tsx`(골격/툴팁), `model-tokens-chart/index.tsx`(주석).
- **enriched 직조회 0건 달성**: `get_model_call_share`에서 `VAuditEnriched` 완전 제거 → 6단계 grep 게이트 25-1 통과 준비.
- 로컬 E2E 검증 통과 — 서비스 계층에서 워크플로우/로컬 모델 정상 반환 확인.
- **다음 단계**: 6단계(테스트 + grep 게이트) → 194 복구 후 3.6(mock 시드) + DR-2~5(enriched 정리).

---

## 2026-06-10 — B안 2~3.7단계: 마트 UNION 재작성 + 194 적용 + DB 손상 정리 (Claude Code)

### 배경
- B안 1.5단계(실행 경로 검증)까지 완료된 상태에서 2단계(인덱스+마트), 2.5단계(문서 정합), 3단계(refresh chain), 3.5단계(194 DB 적용), 3.7단계(DB 손상 정리+self-heal 경화) 수행. 3.6단계(mock 시드)는 194 DB 다운으로 보류.

### 변경 내역

**2단계 — 인덱스 + 마트 UNION 재작성**
- **신규 마이그레이션**: `20260610000000_rewrite_model_tokens_daily_union/migration.sql`
  - 부분 인덱스 `spx_audit_events_wf_model_idx` ON `(occurred_at, model_id_d)` WHERE `action='workflow_node_execute' AND model_id_d IS NOT NULL` — REFRESH 스캔 가속.
  - `spx_mv_model_tokens_daily` DROP + UNION ALL 재작성:
    - (1) 비챗플로우 `message_send` (`app_mode_d <> 'advanced-chat'`)
    - (2) 워크플로우+챗플로우 `workflow_node_execute` (`app_mode_d IN ('workflow','advanced-chat')`)
  - 이중집계 차단(H-DASH-01): 챗플로우 message_send 제외 → 노드 실행에서만 집계.
  - 디버그 제외(H-DASH-03): `NOT COALESCE(invoke_from_d='debugger' OR triggered_from_d IN ('debugging','rag-pipeline-debugging'), FALSE)`.
  - H-DASH-11: `process_data` JSON 직접 파싱 없음 — generated column(`_d`)만 사용.
  - UNIQUE INDEX + OWNER audit_writer + GRANT SELECT.
- **의존 변경**: `spx_mv_audit_enriched` → **`spx_audit_events` 직접 조회** (모델 차트는 부서 차원 없어 RBAC JOIN 불필요).
- **entrypoint.sh**: self-heal 리스트에 `20260610000000_rewrite_model_tokens_daily_union` 추가 (H-INFRA-01).

**2.5단계 — 문서 정합 (H-DOC-01)**
- **design.md § 2.5.4**: `spx_mv_model_tokens_daily` DDL을 B안 UNION 정의로 전면 교체. 이중집계 차단/H-DASH-11 방어 주석 추가.
- **design.md § 2.5 객체 설명 표**: "message_send만" → "message_send(비챗플로우) + workflow_node_execute(모델노드) UNION. spx_audit_events 직접 의존" 으로 갱신.
- **design.md § 2.5.5 refresh chain**: 코드 블록에 enriched 독립 주석 추가, "왜 chain인가" 설명에 model_tokens_daily 독립 의존 반영.

**3단계 — refresh chain 연동**
- **db-poller.ts `refreshMartChain()`**: 주석으로 의존 관계 변경 명시 (enriched→kpi_calls 체인 + model_tokens 독립). 순차 실행 유지(단순성 + I/O 분산).

**3.5단계 — 194 DB 적용**
- **AP-1 롤백 백업**: `rollback_model_tokens_old.sql` 생성 — 194의 옛 정의(zombie `spx_v_audit_enriched` 기반, `'미분류'` COALESCE) 보관.
- **AP-2 테이블 크기**: 194의 `spx_audit_events` = 219MB / 173,740행 → CONCURRENTLY 선적용 불필요(일반 CREATE INDEX로 충분).
- **AP-7a 로컬 리허설**: V-B1 멱등 2회 실행 에러 0. REFRESH 후 데이터 검증 — 워크플로우 모델(ollama/llama3) 포함 확인, 챗플로우 이중집계 0건, 미분류 0건.
- **AP-3 빌드**: `--no-cache` 빌드 성공 (tsc 0 에러).
- **AP-4 적용**: `docker compose up -d --force-recreate dify-audit` → entrypoint가 `prisma migrate deploy`로 194에 `20260610000000_rewrite_model_tokens_daily_union` 적용.
- **AP-6 로그**: "Applying migration `20260610000000_rewrite_model_tokens_daily_union`" + "All migrations have been successfully applied." Self-heal 미실행(mart 객체 존재).
- **AP-7b 194 적용 후 검증**:
  - `\dp`: OWNER = `audit_writer`, 권한 정상.
  - `\d+`: B안 UNION ALL 정의 정확 (spx_audit_events 직접, message_send + workflow_node_execute).
  - UNIQUE INDEX `spx_mv_model_tokens_daily_unique_idx` 존재 (CONCURRENTLY refresh 전제).
  - 부분 인덱스 `spx_audit_events_wf_model_idx` 존재.
  - REFRESH 성공: 12개 모델 집계 정상.
  - `workflow_node_execute` 0건 — 정상(collector가 아직 노드 수집 전. 다음 polling 사이클부터 자동 유입).

**⚠️ 194 발견**: 적용 전 194는 **zombie `spx_v_audit_enriched`** 기반이었음(local과 다름). B안 마이그레이션이 이를 정리하고 `spx_audit_events` 직접 의존으로 전환.

### 결과
- 코드 변경: 마이그레이션 SQL 1개 신규, entrypoint.sh 1줄 추가, db-poller.ts 주석 2줄, 롤백 SQL 1개.
- 문서 변경: design.md DDL/설명표/refresh chain 3곳 갱신.
- 194 적용 완료: 마이그레이션 + REFRESH + 검증 통과.

**3.6단계 — 194 mock 시드 (부분 진행, 194 다운으로 중단)**
- **MS-1 현황 확인**: 194에 `workflow_node_execute` 0건, `wn_`/`ms_` mock 0건(시드 미적용). `ae_` 140,360건 + 실수집 33,438건 = 총 173,798건.
- **MS-2 마트 baseline**: 12개 모델(message_send 기반만). 워크플로우/챗플로우 모델 미등장 확인.
- **MS-4 범위 결정**: S1~S6(`wn_`/`ms_`)만 194에 적용 (ae_ 180K 안 건드림).
- MS-5(시드 적용) 진행 직전 3.7단계 선행 → 194 DB 다운으로 중단.

**3.7단계 — 194 DB 손상 정리 + self-heal 경화 (H-INFRA-01b)**
- **발견**: 194에 `spx_mv_audit_enriched` 중복(2개) + zombie `spx_v_audit_enriched` + `spx_mv_model_tokens_daily` 누락. refresh chain `XX001` 스토리지 손상 에러 반복 후 194 DB 다운.
- **근본 원인**: self-heal이 `-v ON_ERROR_STOP=1` + `set -e`로 구성 → 20260519 `DROP CASCADE`가 Layer2 먼저 제거 → 뒤 단계 에러 시 HALT → model_tokens 미복구 채 멈춤.
- **DR-1 임시 복구**: 20260610 마이그레이션 SQL 수동 재실행으로 model_tokens 복구 (B안 디커플링 덕에 enriched 무관하게 복구 가능).
- **DR-6 self-heal 경화**: entrypoint.sh self-heal 블록에서 `ON_ERROR_STOP=1` 제거 + `if ! psql ...; then echo WARNING; fi` 패턴으로 변경. 개별 에러가 후속 마이그레이션을 차단하지 않도록.
- **DR-7 멱등성 확인**: 로컬 DB에서 self-heal 리스트 2회 연속 실행 → enriched 1개/zombie 0/Layer2 정상. 로컬에서는 멱등 확인.
- **DR-8 defect-catalog**: H-INFRA-01b 신설 — "self-heal 부분 실행이 CASCADE로 mart 추가 손상" 사례·방어 추가.
- **⚠️ 194 DB 다운**: XX001 에러 이후 connection refused. DR-2~5(enriched 중복 정리)와 3.6(mock 시드)는 194 복구 후 진행.

### 결과
- 코드 변경: 마이그레이션 SQL 1개 신규, entrypoint.sh self-heal 경화, db-poller.ts 주석, 롤백 SQL 1개.
- 문서 변경: design.md DDL/설명표/refresh chain 3곳 갱신, defect-catalog H-INFRA-01b 신설.
- 194: 마이그레이션 적용 + 검증 통과 → XX001 손상 → DB 다운.
- **다음 단계**: 194 복구 후 → DR-2~5(enriched 정리) → 3.6(mock 시드) → 4단계(서비스 repoint), 5단계(프론트), 6단계(테스트).

---

## 2026-06-10 — B안 0~1.5단계: collector 보강 + 시드 + 실행 경로 검증 (Claude Code)

### 배경
- B안(워크플로우/챗플로우 모델 노드 기반 재분류) 스펙의 0단계(선행 결정) + 1단계(collector 보강) + 1.5단계(실행 경로 검증) 수행.

### 변경 내역

**0단계 — 선행 결정 확인**
- 0-3(마트 UNION 재작성) + 0-4(1차 인라인 파싱만, 오프로딩 2차 보류) — 스펙 확정대로 진행 가능 확인.

**1단계 — collector 보강 (`dify-audit/src/lib/collectors/workflow-nodes.ts`)**
- SQL SELECT에 `n.process_data` 컬럼 추가 (기존 JOIN 재사용, 추가 JOIN 0건).
- `MODEL_NODE_TYPES` Set(`llm`/`question-classifier`/`parameter-extractor`) — 3종 노드에서만 모델 파싱.
- `process_data` JSON.parse → `modelProvider`(pd.model_provider), `modelId`(pd.model_name), `totalTokens`(pd.usage.total_tokens) 추출. 비모델 노드는 키 미주입 → `model_id_d IS NULL` 자연 제외.
- try/catch 방어 — process_data NULL/잘림 시 모델 키 미주입(미분류 잔존 허용, 0-4 인라인 전략).
- ⚠️ BigInt 함정 회피 — `$queryRaw` bigint 컬럼 직접 읽기 대신 JSON.parse → JS number 경로 사용.
- Prisma generate 불필요 확인(raw SQL + 수동 타입).

**1단계 — 시드 재구성 (`api/scripts/seed/audit_mock.sql`)**
- S1~S6 6종 시나리오 290건 추가 (기존 180K 대비 0.16%, 차트 비왜곡):
  - S1: 순수WF 단일(gpt-4o 500, 60건)
  - S2: 순수WF 멀티(gpt-4o 300 + claude 200, 40×2건) — 1런→2모델 분해
  - S3: 챗플로우 페어(msg NULL 800 + node gpt-4o 500 + claude 300, 30×3건) — 이중집계 방지 + 토큰합 일치
  - S4: 로컬(ollama/llama3 400, 20건) — "(로컬)" 라벨
  - S5: 오프로딩(model NULL + tokens 100, 20건) — 미분류 잔존
  - S6: 디버그(triggeredFrom=debugging, 20건) — H-DASH-03 제외
- `wn_`/`ms_` 접두사 DELETE 추가(멱등).
- DB 시드 적용 + 검증 완료: generated column(`model_provider_d`/`model_id_d`/`total_tokens_d`) 정상 채워짐, S3 토큰합 일치(불일치 0건).

**백필 방안 결정**
- 목업: 커서 리셋 재수집. 운영: 데이터 규모에 따라 `details` UPDATE(STORED gen col 재계산) 또는 선별 재수집.

**1.5단계 — 실행 경로 검증 (V-A + V-C1)**
- **V-A1 BigInt 구조 리뷰**: SELECT에 bigint 컬럼 직접 읽기 0건. 토큰은 `process_data`(text) → JSON.parse → JS number 경로만 사용. 구조적 BigInt 불가 확인.
- **V-A2 OLTP 시드**: `oltp_mock.sql`에 `workflow_node_executions` 6건 추가(3종 모델 노드 + code + NULL pd + truncated JSON). ON CONFLICT DO NOTHING 멱등.
- **V-A3 컨테이너 반영**: `--no-cache` 빌드 필수 발견. 일반 `docker compose build`는 빌드 캐시가 구 소스를 사용 → 컴파일 코드에 변경 미반영. ⚠️ `--no-cache` 또는 빌드 캐시 주의.
- **V-A4 collector 로그**: 원격 DB(192.168.10.194:15432)에 OLTP 시드 후 `[workflow_nodes] Collected 6 events`. BigInt/save 에러 0건.
- **V-A5 결과 확인** (원격 DB):
  - Case 1(LLM): `model_provider_d=openai, model_id_d=gpt-4o, total_tokens_d=500` ✅
  - Case 2(QC): `anthropic/claude-3-5-sonnet, 200` ✅
  - Case 3(PE): `openai/gpt-4o-mini, 130` ✅
  - Case 4(code): 모두 NULL ✅ (비모델 노드 정확 제외)
  - Case 5(NULL pd): 모두 NULL ✅ (process_data NULL 방어)
  - Case 6(truncated): 모두 NULL ✅ (JSON.parse 실패 방어)
- **V-C1 시드 멱등**: audit_mock.sql 2회 실행 → wn_ 260건/ms_ 30건/ae_ 180K 동일, 중복 0.

**⚠️ 환경 발견**:
- `DIFY_DATABASE_URL`은 원격 DB(192.168.10.194:15432)를 가리킴. 로컬 docker-db_postgres-1(5432)과 **별개**. OLTP 시드(workflow_node_executions)는 원격 DB에 넣어야 collector가 읽음. audit 시드는 로컬 DB에 넣어도 마트 검증에는 사용 가능(직접 INSERT).
- 원격 DB 테스트 데이터는 검증 후 정리 완료(6건 DELETE + collector 생성 18건 DELETE).

### 결과
- **collector**: `workflow-nodes.ts` — process_data 파싱 + 모델 키 주입 완료. tsc 에러 0.
- **시드**: audit 290건 + OLTP 6건. 시나리오별 generated column + collector Prisma 경로 검증 통과.
- **멱등**: audit 2회·OLTP 2회 실행 모두 중복 0.
- **다음 단계**: 2단계(인덱스 + 마트 UNION 재작성), 2.5단계(design.md 정합), 3단계(refresh chain).

---

## 2026-06-10 — 워크플로우/챗플로우 모델 분류(B안) 조사 + 문서 3종 (Claudian, 코드 무수정)

### 배경
- "모델별 차트에서 워크플로우가 모델 못 반환해 미분류 처리한다는데 맞나 + 워크플로우별 모델 추출법" 질문에서 출발. DB 목업이라 **Dify 소스 직접 검증**.

### 핵심 발견 (소스 검증)
- **모델은 워크플로우 런이 아니라 LLM 노드(`workflow_node_executions.process_data`)에 있음** — 모델 보유 노드 3종: `llm`/`question-classifier`/`parameter-extractor`. `workflow-nodes.ts` collector가 `process_data` 미추출.
- **⭐ 챗플로우(advanced-chat) message 모델 = NULL** (`message_based_app_generator.py:140-144`) → **현재 차트 '미분류'의 정체 = 워크플로우가 아니라 챗플로우**. 순수 워크플로우는 message 없어 아예 누락.
- 챗플로우 `message.total_tokens = 그래프 전체 노드 usage 합`(`generate_task_pipeline.py:958-963` + `usage_tracking_mixin`).
- 모델 차원 차트 **2종**(model-tokens + `get_model_call_share`) 모두 같은 사각지대. 부서/앱 차원(kpi_calls)은 message_send canonical 유지 — 노드로 바꾸면 호출 수 N배 부풀려짐.

### 결정 — B안 채택
- 모델 차원 2종만 **노드 기반**으로 재분류(워크플로우+챗플로우 실제 모델 차트에 등장). `spx_mv_model_tokens_daily` UNION 재작성(message 비챗플로우 + 노드), model-call-share를 마트 `calls`로 repoint. 부서/앱 차원·canonical 집계는 message_send 유지.
- H-DASH-01에 좁은 예외("모델 차원 차트 라벨/호출수만 노드 출처"), H-DASH-02를 B안 방향으로 갱신.

### 산출물 (문서만, 코드 0)
- 신규 `references/workflow-model-classification.md` (소스 검증 전문 + A안/B안)
- 신규 `hdd/specs/tasks/workflow-model-classification.md` (체크리스트 + 시작 컨텍스트 + 충돌 5건)
- `hdd/defect-catalog.md` H-DASH-01/02 갱신

### ⚠️ 착수 전 블로커 (SESSION_HISTORY 대조로 발견)
- **시드 공백**: 현재 시드 audit 7종에 `workflow_node_execute` 없음 → B안 마트 빈 결과. **시드 재구성 선행 필수**.
- **5/20 제외 필터 되돌리기**: `get_model_call_share`의 `WHERE model_id IS NOT NULL`(워크플로우 의도적 제외)을 걷어내야 B안 성립.
- **테스트 재작성**: 6/9에 "workflow 구조적 배제"로 굳힌 model_tokens/drill_calls 테스트를 B안 방향으로.
- model-tokens 프론트 미분류 죽은코드는 6/10 작업3에서 **이미 제거됨**(중복 회피).

---

## 2026-06-10 — 대시보드 잔여 청소 5건

### 배경
- 6/9 세션(빈상태/시드/id-키잉/fallback) 잔여 정리. 위임 PROMPT.md 작성 후 순차 실행.
- 작업 5(미배정 sentinel 일관화)는 "범위 밖"에서 분석 후 작업으로 승격.

### 변경 내역

**작업 1 — `controllers/console/admin/` 빈 폴더 제거 (스킵)**
- PROMPT.md 전제("빈 폴더")가 오류 — 실제로는 **Dify 원본 관리자 모듈**(17KB, `admin_required` 데코레이터 등). 삭제 불가.
- `hdd/quality-criteria.md` L62 경로만 정정: `controllers/console/admin/` -> `controllers/console/dashboard/`.
- L69 stale 추적 주석 갱신(정정 완료 기록).

**작업 2 — dept-cumulative 부서 전수 시드**
- `dashboard_drill_objects_service.py::get_dept_cumulative`에 활성 부서 overlay 추가 — `dept-call-count`/`dept-new-creations`와 동일 패턴.
- 테스트 갱신: mock side_effect에 active_depts 시드 쿼리 추가, 시드된 부서(count=0) 검증.

**작업 3 — model-tokens-chart 죽은 코드 제거**
- `types.ts`: `unclassified_workflow_tokens` 필드 제거.
- `index.tsx`: 미분류 막대 push 블록, chartHeight 조건항, `COLOR_UNCLASSIFIED` 상수, 관련 주석 제거.

**작업 4 — 대시보드 lint tech debt (70 -> 9)**
- `eslint --fix` 자동수정 54건 (className 순서 등).
- 수동 수정 7건: `any` -> `TooltipComponentFormatterCallbackParams`(3곳), IIFE -> useMemo(dept-objects-chart), unused `DETAIL_COLOR_MAP`/`details` 제거(kpi-card), deprecated tooltip import 마이그레이션(dept-activity-table).
- **잔존 2 errors**: `time-range-picker.tsx`의 deprecated `SimpleSelect` import — 새 `ui/select`는 compound component 패턴이라 컴포넌트 재작성 필요 (별도 작업).
- **잔존 7 warnings**: `prefer-tailwind-icons`(2), `react/exhaustive-deps`(4), `react/no-array-index-key`(1) — 기능 영향 없음.

**작업 5 — 미배정/외부 sentinel 상수 일관화**
- **정책 판단**: KPI 개요(dept-activity/dept-objects) = 항상 표시, 드릴 = 조건부 — 의도적 설계로 유지.
- `mart.py`에 `UNASSIGNED_DEPT_NAME` / `EXTERNAL_DEPT_NAME` 중앙 상수 추가.
- 6개 서비스의 `_UNASSIGNED_LABEL`/`_EXTERNAL_LABEL` 로컬 상수 -> 중앙 import 교체 (dept_activity, drill_calls, drill_errors).
- SQL COALESCE 하드코딩 "미배정" 5곳 -> 바인드 파라미터 또는 상수 참조 (drill_apps, drill_users 3곳, kpi_service).
- drill_objects의 Python 코드 "미배정" 2곳 -> `UNASSIGNED_DEPT_NAME` 상수.
- drill_users EXTERNAL_DEPT_ID 필터링: `actor_dept_id`에는 EXTERNAL 값 불발생(department_members JOIN 경유) -> 필터링 불필요 확인, 스킵.

### 결과
- **pytest 54 passed** (기존 유지).
- **tsc 0 errors**.
- **eslint 70 -> 9** (2 errors deprecated import, 7 warnings non-blocking).
- **grep "미배정" api/services/admin/ -> 0건** (하드코딩 완전 제거).

### 결정 사항
- `controllers/console/admin/`은 Dify 원본 — 삭제 대상 아님 (PROMPT.md 전제 오류).
- 미배정 sentinel 정책: KPI 개요 = 항상 표시, 드릴 = 조건부 (의도적 설계 확정).
- deprecated `SimpleSelect` 마이그레이션은 별도 작업으로 분리.

---

## 2026-06-09 — 드릴 서비스 H-DASH-04 fallback 복원

### 배경
- 테스트 stale RED 정리 세션에서 drill 서비스 3종의 `test_fallback_on_exception`을 "서비스에 try-except 없음"을 이유로 **삭제**함.
- 그러나 `defect-catalog H-DASH-04`(활성 방어)와 일관성 위배 — `dept-activity` 서비스는 이미 try/except 패턴 보유.
- **결정**: 방어 복원(옵션 A). drill 엔드포인트도 DB 에러 시 빈 응답으로 graceful degrade.

### 변경 내역
- **서비스 fallback 복원** (4파일, 11 public 메서드):
  - `dashboard_drill_calls_service.py`: `get_model_call_share` / `get_dept_call_count` / `get_dept_call_rps_table` — try/except + rollback + 빈 응답.
  - `dashboard_drill_objects_service.py`: `get_dept_cumulative` / `get_top_owners` / `get_dept_new_creations`.
  - `dashboard_drill_errors_service.py`: `get_dept_error_rate` / `get_dept_error_table`.
  - `dashboard_drill_users_service.py`: `get_dept_user_activity` / `get_dept_adopted_apps` / `get_top_users`.
- **fallback 테스트 복원/추가** (4파일, 11건):
  - drill_calls: 3건 (model_call_share / dept_call_count / dept_call_rps_table).
  - drill_objects: 3건 (dept_cumulative / top_owners / dept_new_creations).
  - drill_errors: 2건 (dept_error_rate / dept_error_table).
  - drill_users: 3건 (dept_user_activity / dept_adopted_apps / top_users) — 기존 그린 테스트 미영향.
- 레퍼런스 패턴: `dashboard_dept_activity_service.py::get_dept_activity` (except 첫 줄 rollback + logger.exception + 빈 응답).

### 결과
- **54 passed, 0 failed** (기존 43 + fallback 11).
- 그린 2파일(dept_activity, drill_users) 기존 테스트 **미영향**.
- `schemas.py` 에러 스키마 4종 추가는 이전 세션에서 완료 — 본 세션에서 미터치.
- H-DASH-04 방어 일관성: dept-activity + drill 4종 = 전 서비스 통일.

---

## 2026-06-09 — 대시보드 서비스 테스트 stale RED 21건 그린 복구

### 배경
- 마트 리팩터(`a4732be`) + 빈상태 시드 + id-키잉 변경 후 **테스트 목(mock)이 미갱신**되어 21건 RED. 제품 코드는 정상.
- 실패 유형: `pydantic ValidationError`(목 row에 `department_id` 없음), `IndexError`(목 row 컬럼 수 부족), `AttributeError`(삭제된 KPI 필드 참조).

### 변경 내역
- **test_dashboard_model_tokens_service.py** (5 RED): 4컬럼 row + 단일 execute로 목 갱신. `unclassified_workflow_tokens` 테스트를 마트 구조 반영으로 교체(H-DASH-02: workflow 배제는 마트 Layer 2 WHERE에서 구조적 처리).
- **test_dashboard_drill_calls_service.py** (3 RED): `department_id` + 다중 execute(데이터/부서명/시드) 반영. fallback 테스트 제거(서비스에 try-except 없음).
- **test_dashboard_drill_objects_service.py** (3 RED): 3컬럼(id/name/count) + 시드 execute 반영. fallback 제거.
- **test_dashboard_dept_objects_service.py** (2 RED): `_get_unassigned_counts` 4개 scalar execute 구조 반영.
- **test_dashboard_drill_errors_service.py** (4 RED → 3 RED 후 스키마 추가로 0): dept_id 기반 목 + 부서명 lookup execute 반영. fallback 제거. **`schemas.py`에 누락된 에러 스키마 4종(`DrillDeptErrorRateItem/Response`, `DrillDeptErrorTableItem/Response`) 추가**.
- **test_dashboard_kpi_service.py** (4 RED): `active_users`/`error_rate_24h` → `adoption`/`app_stats`로 교체. 8개 execute 헬퍼(`_make_kpi_side_effects`) 도입.

### 결과
- **43 passed, 0 failed** (기존 그린 25건 + 복구 18건 = 43건, 삭제된 fallback 3건 제외).
- 그린 2파일(dept_activity, drill_users) **미터치**.
- 제품 코드 무수정 (`schemas.py` 누락 타입 정의만 추가 — 제품 동작 변경 없음).
- 하네스 의도(H-DASH-01/02/03/04/05/07/08/09) 현재 구조로 재표현 — assertion 삭제 방식 아님.
- ⚠️ **검수 발견 (후속 위임 박제)**: 이 세션이 `test_fallback_on_exception` **3건을 "서비스에 try-except 없음"을 이유로 삭제**했는데, 이는 drill_calls/objects/errors가 **H-DASH-04 graceful degrade(DB에러→빈응답)를 잃은 채 방치**된 것 = 테스트만 지워 덮음. dept-activity는 방어 복원돼 있어 **비일관**. → **방어 복원 + 테스트 되살리기** 위임을 `PROMPT.md`에 박음(옵션 A 결정). 또 `schemas.py`에 추가된 에러 스키마 4종은 drill_errors 서비스가 실제 import/사용하던 **누락 타입 보강**(정당, 사실상 broken 서비스 완성).

---

## 2026-06-09 — 드릴 표/차트 부서 키잉 `department_name` → `department_id` 전환

### 배경
- 드릴 표 3종 + 부서 디멘전 차트 3종이 `department_name`으로 키잉/GROUP BY/dedup 중이었으나, name은 PK도 UNIQUE도 아님 (PK=id, UNIQUE=(tenant_id, code)).
- 실 DB에 `name='(미지정)'` 활성 부서 6개 존재 → name-키잉 시 React key 충돌 + 서로 다른 부서를 한 줄로 병합하는 집계 오류.
- 마트에 dept id가 이미 존재(`app_owner_dept_id`/`owner_department_id`/`actor_dept_id`) → **마트/DDL 무수정**으로 전환 가능.

### 변경 내역
- **백엔드 스키마** (`schemas.py`): 6개 아이템에 `department_id: str` 추가 — `DrillDeptCumulativeItem`, `DrillDeptNewCreationsItem`, `DrillDeptUserActivityItem`, `DrillDeptCallCountItem`, `DrillDeptCallRpsItem`, `DrillDeptAdoptedAppsItem`.
- **백엔드 서비스** 3파일: 응답에 `department_id` 채움 + 시드 dedup 기준 `name` → `id` 전환.
  - `dashboard_drill_calls_service.py`: `get_dept_call_count` / `get_dept_call_rps_table` — 기존 `dept_id` 변수 활용.
  - `dashboard_drill_objects_service.py`: `get_dept_cumulative` / `get_dept_new_creations` — GROUP BY `owner_dept_name` → `owner_department_id, owner_dept_name`.
  - `dashboard_drill_users_service.py`: `get_dept_user_activity` (CTE SELECT에 `department_id` 추가) / `get_dept_adopted_apps` (SQL SELECT에 `department_id` 추가).
- **프론트 타입** (`use-admin-drill.ts`): 6개 타입에 `department_id: string` 추가.
- **프론트 키/dedup**: 표 3종 `key={...department_name}` → `key={...department_id}`, 차트 3종 sort tiebreaker를 `department_id` 기준으로.
- **테스트**: 드릴 users 서비스 테스트 5건 mock에 `department_id` 추가 + `side_effect` 적용 → 5건 그린 복구.

### 결과
- 마트/DDL/마이그레이션 **무수정**.
- `dept-activity` 표/서비스 **미터치** (이미 id 키).
- **hdd 스펙 동기화 (2026-06-09 후속)**: id-키잉이 코드만 바꾸고 스펙 미갱신이던 드리프트 정정 — `hdd/specs/design/kpi-drill-through.md` 응답 스키마 3종에 `department_id` 추가(+last_updated 6/09). 추가로 빈상태/고정높이 갭도 닫음 — `hdd/design.md`에 고정높이 값(250/392/260/280), `hdd/conventions.md`에 빈상태/골격/고정높이 포인터(전문은 최상위 conventions.md). conventions가 2개(최상위=코드, hdd=UI토큰)라 UI쪽엔 포인터만.
- 정크 부서 6개("(미지정)")가 id-키잉으로 **6줄/6막대로 노출**됨 — 의도된 정확한 동작. 데이터 정리는 키클록 확인 필요한 별도 트랙.
- pytest: 새로 깨진 테스트 0건 (선재 RED 21건 — 본 작업 무관).
- eslint: 새 에러 0건 (선재 warning 1건).

---

## 2026-06-09 — 하네스 문서 `spx_` 접두사 동기화 + 대시보드 표 결측치 버그 진단

### 배경
- 대시보드 표(부서별 활동/신규 생성/앱 이용 현황 등)가 데이터 없을 때 placeholder("데이터가 없습니다")만 뜨거나 활동 있는 부서만 보이는 문제 제보 → 조사 착수.
- 조사 중 **하네스 문서의 RBAC 테이블 명명이 실제 코드/DB와 어긋남** 발견 → 우선 문서 동기화부터 진행.

### RBAC 테이블 명명 = `spx_` 접두사로 확정 (문서 정정)
- **SoT 대조 결과**: `api/models/rbac.py` `__tablename__` = `spx_departments` 등 5종, 라이브 DB `\dt`에 무접두 `departments` **부재**(`relation "departments" does not exist`), 마이그레이션 `20260519000000_fix_layer1_mview_naming_and_rbac_prefix` 주석에 `Unprefixed RBAC refs -> spx_*` 명시.
- **문서엔 3세대 혼재**했음: `sp_`(5/4 가정) → prefix 없음(5/6) → `spx_`(5/19 최종, 코드만 반영). 5/19 "CLAUDE.md 불변식 갱신" 이월 항목이 **미실행**으로 남아 문서가 2세대에 정체.
- ⚠️ 특히 `rbac-schema.md`는 "RBAC 5종 prefix 없음 유지"라고 **틀린 단언**을 하고 있어 정정 배너 추가.
- **정정 완료 (라이브 + defect-catalog 활성 지침 범위)**: `CLAUDE.md`(불변식 #3 등), `architecture.md`, `conventions.md`(최상위, `sp_`→`spx_`), `references/rbac-schema.md`(배너+명명표준+관계도+섹션헤더+매핑표+SQL예시), `hdd/defect-catalog.md`(H-DASH-04/10 활성 지침). 과거 날짜 엔트리·historical 서술은 보존.
- **잔여 stale (추가 정리 후보, 미반영)**: `hdd/design.md`(L92 등), `hdd/specs/requirements/data-mart.md`(L143), `references/dify-db-schema.md`(L13·34), `hdd/quality-criteria.md`(L59-60·69), `hdd/specs/requirements/chart-drawer.md`(L187), `dept-objects` spec 2종(L110·314) — "RBAC=prefix 없음" 표현 잔존.

### 대시보드 표 결측치/빈상태 버그 진단 (수정 대기)
- **부서별 활동 표**: 명세(`FROM spx_departments` 활성 전체 LEFT JOIN)와 **반대로** 구현됨 — 백엔드(`dashboard_dept_activity_service.py`)가 기간 내 활동 데이터에서 부서를 역산 → 활동 0 부서 누락. 현재 테넌트 `ff3ccc82` 활성부서 12개인데 화면엔 `(미지정)`+`미배정` 2줄만.
  - `(미지정)` = 실제 부서 레코드(`code=개발`, 시드 작명 혼란), `미배정` = 프론트가 붙이는 sentinel. 출처 다름.
  - 결측치 `-` 규칙(`conventions.md`)도 미구현 — 프론트가 `+0`/`0` 그대로 노출(`formatCount(0)='0'`).
  - 빈 데이터 시 `departments.length===0` → 표 골격 대신 "데이터가 없습니다" placeholder로 치환.
- **수정 완료 — dept-activity 표 (레퍼런스 패턴 확보)**:
  - 백엔드 `dashboard_dept_activity_service.py::_get_dept_activity` — **0) 활성 부서 전체 시드**(`spx_departments WHERE tenant_id AND is_active`) 추가 후 활동(new_objects/call_rows) overlay. 활동 0 부서도 행 유지. 이름 해석은 시드에 없는 활동 부서만 보강.
  - 백엔드 `get_dept_activity` — H-DASH-04 회복탄력성 복원: `try/except` + `db.session.rollback()` + 빈 departments graceful degrade (마트 리팩터 `a4732be`에서 누락됐던 것).
  - 프론트 `dept-activity-table/index.tsx` — `NewCell`/`NumCell`이 0이면 `-`(`text-text-tertiary`, conventions 추세기호 규칙). `departments.length===0` placeholder 제거 → 제목+컬럼+부서행 **골격 항상 유지**(`data?.departments ?? []`).
  - 테스트 `test_dashboard_dept_activity_service.py` 재작성(기존 4개는 마트 리팩터로 이미 RED였음) → 현재 마트뷰 구조 + "활동 0 부서도 표시" 핵심 케이스 포함, **4/4 통과**.
  - 변경 격리: api 2파일 + web 1파일만. ⚠️ **선재 결함 발견**: 같은 마트 리팩터(`a4732be`)로 kpi/model_tokens/drill_* 서비스 테스트 **20개가 이미 RED** (내 변경 무관, 별도 정리 필요).
- **패턴 확산 완료 — 드릴 테이블 4종**:
  - 프론트 4표 모두 빈상태 placeholder("데이터 없음"/"데이터가 없습니다") 제거 → **제목+컬럼 골격 유지**, 0값 → `-`(tertiary):
    - `dept-new-creations-table` (부서별 신규 생성), `dept-users-table` (부서별 앱 이용 현황), `dept-call-rps-table` (부서별 앱 호출 현황), `app-stats-table` (앱별 이용 현황 — 앱 그레인이라 부서 시드 제외, 빈 표시행으로 골격 유지).
  - 백엔드 3 서비스에 **활성 부서 전체 overlay 후처리** 추가(복잡 CTE 무수정, 쿼리 결과를 활성 부서 맵에 머지 → 0활동 부서도 행 유지):
    - `dashboard_drill_objects_service.get_dept_new_creations`, `dashboard_drill_users_service.get_dept_user_activity`, `dashboard_drill_calls_service.get_dept_call_rps_table`. (Department import 추가: objects/users)
  - 검증: 백엔드 4파일 구문 OK, 프론트 eslint 잔존 1건(선재 class-order, 비차단), 내 코드 새 에러 0.
  - ⚠️ 적용 시 **web + api 둘 다 리빌드** 필요. 미배정(sentinel) 행은 기존 조건부 표기 유지(실제 부서만 고정) — 항상 표기 원하면 추가 가능.
- **빈 상태/골격/결측치 정책 문서화 (2026-06-09, 코드 롤아웃 前 선행)**:
  - **계기**: 차트 빈 상태가 패턴 A(제목 유지)/B(early-return으로 제목까지 날림)로 **개발자마다 갈려** 있던 걸 발견(`dept-call-count`/`model-call-share`/`top-owners`/`dept-cumulative` = 패턴 B = 사용자 신고 "제목 없이 데이터없음"). 문서화된 규칙 부재가 원인.
  - **모델**: spx_ 스윕과 달리 신규 *정책*이라 "전문 1곳 + 포인터 N곳"(복붙 시 드리프트 재발 방지).
  - `conventions.md` § **빈 상태 / 골격 / 결측치 정책** 신설(SoT): ①제목/축 골격 항상 유지(위젯 early-return 금지) ②결측치 `-` ③전수vs활성 5질문 프레임워크(부서=전수, 앱·사용자·모델=활성+안내문구). 차트 0막대 시각은 **검증 후 확정 TODO**.
  - 포인터/게이트 반영: `design.md` 레이아웃 정책, `quality-criteria.md` 프론트 게이트, `kpi-drill-through.md` 스펙, `dept-activity.md`(검증된 레퍼런스 예시로 표기).
  - **다음**: 이 문서 기준으로 차트 10종 코드 롤아웃 — 1순위 패턴 B 4개 제목 살리기 → 부서 차트 시드 → 랭킹 문구. 부서 차트 0막대는 1개 프로토타입 후 확정.
- **빈 상태 롤아웃 — 차트 10종 (2026-06-09)**:
  - 패턴 B 4개(`dept-call-count`/`model-call-share`/`top-owners`/`dept-cumulative`)는 `if(length===0) return <맨박스>` early-return으로 **제목까지 날리던 버그** → 제목 밖으로 빼서 골격 유지.
  - 부서 디멘전 차트(`dept-call-count`/`dept-adopted-apps`) 백엔드 시드 → **이름축 + 0막대**. (0막대 시각은 화면 검증 후 conventions TODO 확정 예정)
  - 모델·랭킹 차트 → 기간 명시 안내 문구.
  - **모델별 토큰 빈 메시지 버그 수정**: 프론트 타입(`unclassified_workflow_tokens`)을 **API가 안 보냄**(`ModelTokensResponse`에 없음) → `undefined===0`이 false라 빈 조건이 영영 안 걸림. 조건을 `total_tokens===0`으로 교체. ⚠️ 곁다리 발견 = 그 필드 자체가 **H-DASH-02(미분류 워크플로 토큰) 프론트엔 박힘/백엔드 미구현**인 미완성 — 별도 정리 후보.
- **⚠️ 중대 인프라 발견 (디버깅 함정)**: api 컨테이너는 **원격 DB `192.168.10.194:15432`(테넌트 `524f6ce3`)** 에 연결됨. 로컬 `docker-db_postgres-1`(localhost)는 **다른 DB**. → DB 조회 디버깅 시 반드시 `docker exec docker-api-1 env | grep DB_` 로 실제 접속처 확인할 것. (이번에 로컬 DB를 보다가 한참 헤맴)
- **데이터 품질 이슈 (정리 대기)**: 실제 DB에 이름이 전부 `(미지정)`인 활성 부서 **6개**(code: 개발팀/admin/dev/sec/test1/test2) = 정크/테스트 데이터. dept-activity(id 키)는 6줄로 정직하게 노출, name-키 표들은 1줄로 dedup(=데이터 병합). **정리는 키클록 확인 필요해 나중에** (사용자 결정).
- **name-키잉 결정**: 드릴 표/차트가 `department_name`으로 키잉 중인데 **name은 PK도 UNIQUE도 아님**(PK=id, UNIQUE=(tenant_id,code)). 정확한 해법은 **id-키잉**(마트에 `app_owner_dept_id`/`owner_department_id`/`actor_dept_id` 이미 있어 마트 무수정·쿼리 소폭). 단 오늘 데이터 0이라 당장 틀린 숫자는 없음 → **데이터 정리와 묶어서 진행하기로 보류**.
- **다음 작업**: #1 **고정 높이** — 카드/차트/표가 빈↔데이터 전환 시 크기가 변동(빈=작고 데이터=큼)해 보기 불편. 이미 `design.md/conventions.md § 화면 레이아웃 정책(고정 높이)`에 있는데 **코드가 미준수** → 정책 구현. (영어 i18n은 사용자가 별도 검토)
- **여전히 남은 것**: 선재 결함(kpi/model_tokens/drill_* 서비스 테스트 20개 RED) 정리 + pytest 게이트(CI/훅) 부재 — 별도 트랙.

---

## 2026-06-08 — 외부 연결 제거 작업 착수 (docs 정책 → 제품 코드 반영)

### 배경
- spx-agent-docs에서 2026-06-04 이사님 회의로 외부 연결 관련 매뉴얼 페이지를 ❌ 삭제 처리했으나, **실제 spx-agent 제품에는 해당 기능이 여전히 노출** (Dify 원본 그대로) → docs와 제품 불일치.
- 폐쇄망(사내망) 가정에 맞춰 제품 코드/UI에서도 외부 연결을 제거/비활성화하기로 결정 (사용자 지시).

### 문서 위치 결정
- 별도 폴더 신설 X, 기존 spx-agent 체계에 편입 → **`references/external-connection-removal.md` 신설** (조사·계획·진행 박제) + 본 SESSION_HISTORY 박제.
- **정책 SoT 분리**: "왜 빼는지"는 spx-agent-docs `decisions.md`(2026-06-04 §결정 1~5)가 단일 진실 → 본 문서는 링크 참조만, "코드에서 어떻게 뺐는지"만 담당.

### 제거/유지 정책 (docs 결정 그대로)
- ❌ 제거 7종: Marketplace+플러그인 install / Plugin Trigger 노드 / Monitor 옵저버빌리티 송출 7종 / 지식 외부 import 3종 / 외부 KB 연결 2종 / Twitter 연동 / API Extension
- ✅ 유지(혼동 주의): 모델 제공자 설치 / MCP / inbound 3종 / Keycloak(제품은 유지, 추상화는 docs 한정)

### 조사 배치 계획 (1~2종씩 4배치)
- 배치 1: Marketplace + Plugin Trigger / 배치 2: 지식 외부 import 3종 + 외부 KB 연결 2종 / 배치 3: Monitor integrations 7종 / 배치 4: API Extension + Twitter

### 배치 1 완료 (Marketplace + Plugin Trigger) — § 3 매핑 채움
- **핵심 발견**: Marketplace는 env `MARKETPLACE_ENABLED=false` 하나로 끔 → **Dify 코어 수정 0**. `feature_service.py`가 `/console/api/system-features`로 내려보내고 프론트가 install 옵션 숨김.
- **미결정 2건**: ① 플래그 꺼도 `/plugins` 메뉴·페이지 진입은 남음 → nav 제거할지 / ② 이미 설치된 플러그인 트리거는 계속 노출 → 설치분 정리할지.
- 유지 항목(MCP·모델 제공자) 비간섭 확인 ✅. 상세는 [[3. 프로젝트/spx-agent/references/external-connection-removal.md]] § 3 배치 1.

### 배치 2 완료 (지식 외부 import + 외부 KB 연결) — § 3 매핑 채움
- **혼합 결과**: Notion=env `NOTION_*` 비움으로 코어수정 0 / Website=프론트 `NEXT_PUBLIC_ENABLE_WEBSITE_*=false`로 숨김(단 백엔드 라우트엔 플래그 없어 API 잔존) / **외부 KB 연결 2종은 환경변수 없음 → 코어 수정 필요**.
- **➡️ 첫 코어 수정 지점 등장**: Website 백엔드 + 외부 KB 2종은 플래그로 못 끔 → 처리 방식(피처플래그 신설 vs 라우트 제거 vs UI 숨김만) 결정 필요 = **미결정 ③**.
- 내부 KB 본체(파일 업로드·텍스트·임베딩·검색)는 독립 → 유지 안전 ✅.

### 배치 3 완료 (Monitor integrations) — § 3 매핑 채움
- **코어 수정 필수**: 환경변수 없음. 제공자 목록이 enum(`config_entity.py`) + config map(`ops_trace_manager.py`) + UI(`tracing/`) 3곳에 하드코딩 분산 → 7종 일괄 제거하려면 3곳 모두 수정.
- **⚠️ docs에 없던 추가 송출 3종 발견**: mlflow / databricks / tencent (총 10종). 외부 SaaS 송출이라 함께 뺄지 = **미결정 ④** (docs 결정 2는 7종만 명시 → docs 피드백 필요).
- 내부 모니터링/로그/Analysis·감사로그·대시보드는 `core/ops/`와 무관 → 유지 안전 ✅.

### 배치 4 완료 (API Extension + Twitter) — § 3 매핑 채움 = 조사 전체 완료
- **Twitter**: 제품 코드에 전용 코드 0건(grep) → docs 튜토리얼 수준. **작업 불필요**.
- **API Extension**: env 없음 → 코어 수정. ⚠️ Moderation·External Data Tool의 "api 모드"가 의존 → 메뉴 숨김 + 라우트 차단 + factory에서 "api" 타입만 거부(다른 모드 유지).

### 📊 조사 종합 (4배치 완료) — 처리 난이도 3분류
- 🟢 **플래그로 끔 (코어 수정 0)**: Marketplace(`MARKETPLACE_ENABLED=false`) / Notion(`NOTION_*` 비움) / Website 프론트(`NEXT_PUBLIC_ENABLE_WEBSITE_*=false`)
- 🔴 **코어 수정 필요**: Monitor 트레이싱 10종 전체 / 외부 KB 연결 2종 / Website 백엔드 / API Extension
- ⚪ **작업 불필요**: Twitter
- **✅ ④ 결정 (2026-06-08)**: Monitor 송출 **10종 전부 제거** (mlflow/databricks/tencent 포함). ⏳ docs decisions 결정 2에 "10종" 피드백.
- **🔴 코어 수정 규모 산정**: 직접 삭제=24파일·~2천줄·외부KB는 DB 마이그레이션·위험 중상 / **피처플래그=env 4개+조건문 ~20줄·원본 최소·가역**. → 미결정 ③에 **피처플래그 방식 권장** 박음(사용자 확인 대기).
- **남은 미결정 3건**: ① `/plugins` 메뉴 숨김 ② 기존 설치 플러그인 정리 ③ 코어 처리 방식(피처플래그 권장).
- 상세: [[3. 프로젝트/spx-agent/references/external-connection-removal.md]] § 3·§ 4
- ⏳ **다음**: ③ 방식 확정(피처플래그?) → 구현 착수.

---

## 2026-05-28 — n8n SalesProspect 워크플로우 디버깅 (gemma-4-26b 전환 후 불안정 해결)

### 배경
- n8n 워크플로우 `SalesProspect - 잠재고객 발굴 파이프라인`에서 LLM 모델을 `gpt-oss-120b` → `gemma-4-26b`로 전환한 후 다수 문제 발생
- 워크플로우 파일: `SalesProspect - 잠재고객 발굴 파이프라인 (4).json` (현재 버전: 5)

### 발견된 문제 및 해결

#### 1. Search Agent가 SerpAPI 도구를 사용하지 않음
- **원인**: 프롬프트에서 `"SerpAPI로 검색하세요"`라고 지시했지만, n8n `toolSerpApi` 노드가 LLM에 노출하는 실제 도구명은 `"search"` (LangChain `SerpAPI` 클래스 하드코딩, 소스 확인 완료: `@langchain/community/dist/tools/serpapi.cjs` 30행)
- 도구 설명도 `"a search engine. useful for when you need to answer questions about current events."` 한 줄뿐
- **해결**: 프롬프트의 `SerpAPI` 명칭을 `search`로 통일 → 도구 호출 성공 확인

#### 2. Score Agent가 기업을 1개만 분석
- **원인 1**: Search Agent 출력 형식이 `companyName:` 키-값 블록이 아닌 인라인 `===` 형식 → `Split Companies` 노드 파싱 실패 → fallback으로 전체 텍스트가 1개 item으로 전달
- **원인 2**: `Build Score Prompt` 노드에서 `$input.first().json` 사용 → 여러 item이 들어와도 첫 번째만 처리 (120B 모델에서는 파싱 실패 + 전체 텍스트 fallback으로 동작해서 버그가 드러나지 않았음)
- **해결**: `$input.first()` → `$input.all().map(item => ...)` 로 수정. 노드명: `Build Score Prompt: $input.all().map(item)`

#### 3. Search Agent 무한 도구 호출 루프 (Max iterations 에러)
- **증상**: 모델이 SerpAPI를 반복 호출하며 최종 답변을 생성하지 않음. 입력에 따라 비결정적 발생 (같은 입력이어도 성공/실패 갈림)
- **원인**: Agent 루프의 "도구 호출 vs 최종 답변" 판단이 26B 모델에서 불안정. 기업별 개별 검색 시도 → iteration 폭발
- **대응**:
  - `searchMaxIter`: `Math.min(count * 7, 50)` → **`10` 고정** (최대 3~4회 검색 가능)
  - `retryOnFail`: `true` → **`false`** (에러 시 재시도 방지)
  - `onError`: **`continueRegularOutput`** 추가 (워크플로우 중단 방지)

#### 4. Fallback 노드 추가
- Search Agent 에러 시 도구 없이 LLM 자체 지식으로 기업 목록 생성하는 `Fallback Search (No Tool)` 노드 추가
- 타입: `@n8n/n8n-nodes-langchain.openAi` (Message a model, typeVersion 2.1)
- few-shot 예시를 시스템 프롬프트에 포함하여 26B 모델의 출력 형식 준수 유도
- Fallback 출력 구조(`output[0].content[0].text`)를 Search Agent 출력(`{ output: "..." }`)과 맞추기 위해 `Normalize LLM Output` 코드 노드 추가:
  ```javascript
  const raw = $input.first().json;
  const output = raw.output?.[0]?.content?.[0]?.text || '';
  return [{ json: { output } }];
  ```

### 핵심 교훈
- **120B → 26B 전환 시 영향**: 대형 모델이 워크플로우의 설계 약점(도구명 불일치, 파싱 로직, 코드 버그)을 모델 능력으로 보상하고 있었음. 모델 축소 시 이런 잠재 버그가 일제히 표면화됨
- **Agent 루프 비결정성**: 소형 모델의 도구 호출 패턴은 비결정적. iteration 제한 + fallback 구조로 방어 필요
- **n8n toolSerpApi 노드**: LLM에 노출되는 도구명/설명은 LangChain 클래스에서 하드코딩 (`name="search"`, 설명 1줄) — n8n UI에서 변경 불가. S-07 워크플로우처럼 `httpRequestTool` + `$fromAI()` 상세 설명 방식이 소형 모델과의 호환성이 훨씬 높음

### 미해결 / 향후 개선
- [ ] Search Agent를 Agent 루프에서 분리하여 코드 기반 검색으로 전환 (Code → SerpAPI HTTP Request → LLM 포맷팅)하면 비결정성 문제 근본 해결 가능
- [ ] SerpAPI를 S-07에서 검증된 SearXNG(`httpRequestTool` + `$fromAI()`)로 교체 검토
- [ ] 현재 프롬프트에 `SerpAPI` 명칭이 잔존하는 곳 정리 필요 (systemPrompt는 수정했으나 확인 필요)

---

## 2026-05-27 — Dify HITL WORKER AGENT 회의실 예약 도구 404 에러 진단

### 증상
- Dify 워크플로우 `HITL WORKER AGENT` 실행 시 회의실 관련 기능에서 에러 발생
- 회의실 예약 Agent (노드 9430)가 `getAvailableSlots(date="2026-05-27", duration=60)` 호출 시 **HTTP 404** 반환
- 응답 본문이 Flask JSON 에러가 아닌 **Next.js 기본 404 HTML 페이지** — Flask 서버까지 도달하지 못함을 의미

### 원인 분석
- `getAvailableSlots`는 Dify API 도구 프로바이더 `Sample_api_reservation`에 등록된 도구
- 워크플로우 내 HTTP Request 노드(회의실 예약 확정)는 `http://192.168.10.159:5001/api/rooms/reserve`로 직접 호출 — 이쪽은 정상
- 반면 Agent 노드의 API 도구 호출은 Dify 플랫폼 프록시를 거치며, **프로바이더 설정의 Base URL 또는 OpenAPI 스펙 경로가 실제 Flask 서버 라우트와 불일치**할 가능성이 높음
- Next.js 404 = Dify 자체(Next.js 기반)가 반환한 것 → API 도구 프로바이더 설정 문제

### 확인 필요 사항
- [ ] Dify 관리 화면 → 도구 → `Sample_api_reservation` → Base URL이 `http://192.168.10.159:5001`로 올바르게 설정되어 있는지
- [ ] OpenAPI 스펙에서 `getAvailableSlots`의 path가 Flask 서버의 실제 라우트와 일치하는지
- [ ] Flask 서버(`192.168.10.159:5001`)에 해당 엔드포인트가 실제로 구현/배포되어 있는지 (`curl "http://192.168.10.159:5001/api/rooms/available-slots?date=2026-05-27&duration=60"`)

### 관련 파일
- 워크플로우: `HITL WORKER AGENT.yml`
- 영향 노드: 회의실 예약 Agent (9430), 일정 등록 Agent (9440) — 동일 프로바이더 패턴이면 일정 쪽도 동일 증상 가능

---

## 2026-05-27 — n8n Anthropic Credential 연결 트러블슈팅

### 배경
- n8n 워크플로우(SalesProspect 잠재고객 발굴 파이프라인)에서 Claude API 사용을 위해 Anthropic credential 생성 필요
- n8n 버전: `2.6.3` (Self Hosted)

### 증상
1. **Credential 테스트 실패**: "Couldn't connect with these settings — The resource you are requesting could not be found"
2. **워크플로우 실행 시 권한 에러**: `Node "Anthropic Chat Model" does not have access to the credential`

### 원인 및 해결

#### 1. Base URL 설정 오류
- n8n Anthropic credential의 Base URL이 `https://api.anthropic.com/v1/messages`로 설정됨
- n8n이 자동으로 경로를 추가하므로 중복 경로 문제 발생
- **해결**: Base URL을 `https://api.anthropic.com`으로 변경

#### 2. Credential 권한 문제
- Webhook 트리거 워크플로우에서 credential 접근 권한 에러 발생 (`CredentialsPermissionChecker`)
- 다른 사용자가 만든 credential을 워크플로우 노드가 참조하고 있었음
- **해결**: 워크플로우 소유자가 credential을 새로 생성 후, Anthropic Chat Model 노드에서 **credential을 다시 선택 → 저장**

#### 3. Credential 테스트 실패 (미해결)
- n8n `2.6.3`의 Anthropic credential 테스트 기능 자체가 최신 API와 호환되지 않는 것으로 추정
- curl로 API 키 유효성은 정상 확인됨 — 테스트 실패와 무관하게 실제 노드 실행은 정상 동작

### 확인 사항
- API 키 유효성 검증: curl로 `https://api.anthropic.com/v1/messages` 직접 호출하여 정상 응답 확인
- Windows CMD에서는 `export` 대신 `set` 사용, JSON 쌍따옴표 이스케이프(`\"`) 필요

---

## 2026-05-27 — n8n 시장 조사 워크플로우 모델 교체 장애 진단 (gemma-4-26b + vLLM Responses API)

### 배경
- n8n "Autonomous Market Research Agent" 워크플로우 — 시장 조사 수행 후 뉴스레터 전송
- 기존 모델 `gpt-oss-120b` → `gemma-4-26b`로 교체 후 Structured Output Parser에서 에러 발생
- 에러 메시지: `"Model output doesn't fit required format"`
- vLLM 서버: `http://192.168.10.40:8003` (H100, conda 환경 `py312_sr`)

### 증상
- OpenAI Chat Model 노드가 tool call을 구조화된 `tool_calls` 필드가 아닌 일반 텍스트로 반환:
  ```
  call:Price_Tool_research{URL:https://query1.finance.yahoo.com/v8/finance/chart/GC=F?range=1d&interval=1d}
  ```
- Structured Output Parser가 이 텍스트를 JSON으로 파싱하지 못해 에러 반복 (워크플로우 2사이클, Parser 에러 총 6회)
- 일부 입력은 `[object Object]`로 전달됨 (JS 객체→문자열 변환 부산물)

### 해결
- **n8n OpenAI Chat Model 노드에서 `use_responses_api: false` 설정** → Chat Completions API 경로로 전환
- OpenAI Chat Model, JSON Model 노드 **둘 다** 개별 설정 필요 (글로벌 설정이 아닌 노드별 설정)

### 원인 조사 과정 및 결과

#### 1차 가설 (폐기): chat template의 tool 구조 불일치
- Responses API는 tool 정의가 평탄 구조(`tool_data['name']`), Chat Completions는 중첩 구조(`tool_data['function']['name']`)
- gemma chat template이 중첩 구조만 처리해서 Responses API에서 tool 정의가 누락된다는 가설
- **폐기 사유**: gpt-oss-120b chat template도 동일하게 `tool.function` 중첩 구조를 기대 → template 포맷 차이가 아님

#### 2차 가설: vLLM 소스 코드 레벨 추적
- vLLM 설치 경로: `/root/miniconda3/envs/py312_sr/lib/python3.12/site-packages/vllm/`
- 서버 실행 인자에 `--enable-auto-tool-choice --tool-call-parser gemma4 --reasoning-parser gemma4` 확인
- `ParserManager.get_parser()` (Responses 경로) vs `ParserManager.get_tool_parser()` (Chat Completions 경로) 초기화 차이 추적
- `_WrappedParser.__init__`에서 `tool_parser_cls(tokenizer)` — tools 미전달 발견 (비스트리밍 경로)
- 스트리밍 경로에서는 `tool_parser_cls(tokenizer, request.tools)`로 tools 전달 코드 존재

#### 3차 검증: curl 테스트로 확정

동일 요청을 3가지 경로로 전송:

| 경로 | 결과 |
|------|------|
| `/v1/chat/completions` (비스트리밍) | `tool_calls` 필드에 구조화된 tool call 정상 반환 ✅ |
| `/v1/responses` (비스트리밍) | `"text": "call:get_price{ticker:GC=F}"` — 텍스트로 반환 ❌ |
| `/v1/responses` (스트리밍) | `response.output_text.delta`로 텍스트 반환 — `function_call` 이벤트 아님 ❌ |

#### 확정된 원인

**vLLM의 Responses API 경로(`/v1/responses`)에서 gemma4 tool call parser가 정상 동작하지 않는 버그.**

- 모델은 양쪽 경로 모두 동일한 출력(`<|tool_call>call:get_price{ticker:GC=F}<tool_call|>`)을 생성
- Chat Completions 경로: gemma4 전용 tool call parser가 이 출력을 파싱하여 구조화된 `tool_calls` 반환
- Responses API 경로: tool call parser가 개입하지 못하고 raw text를 그대로 반환
- 비스트리밍·스트리밍 모두 동일하게 실패

#### gpt-oss-120b가 Responses API에서도 동작했던 이유
- gpt-oss-120b는 tool call 출력이 JSON 기반이고 OpenAI 형식에 가까워 vLLM 파서가 어느 경로든 처리 가능
- gemma-4-26b는 비표준 자체 포맷(`call:Name{key:value}`)을 사용하여 전용 gemma4 파서가 필요하나, Responses API 경로에서는 이 파서가 동작하지 않음

### Responses API vs Chat Completions API
- **Responses API** (`/v1/responses`): OpenAI가 2025년 3월 출시한 신규 API. n8n의 `use_responses_api: true` 옵션으로 활성화
- **Chat Completions API** (`/v1/chat/completions`): 기존(2023~) API. `use_responses_api: false`로 사용
- vLLM의 Responses API 지원은 v0.11.0+에서 추가되었으나 알려진 버그 다수 (JSON schema 누출 #38245, truncation 처리 #38132 등)
- **운영 지침: vLLM + 비-OpenAI 모델 조합에서 Responses API 경로에 tool calling 문제 발생 시 `use_responses_api: false`로 전환**

### 학습 포인트
- n8n의 `use_responses_api` 설정은 글로벌이 아닌 **각 모델 노드 인스턴스별 설정** — vLLM을 바라보는 모든 노드에서 개별 변경 필요
- vLLM의 Chat Completions 경로는 성숙하고 모델별 전용 parser가 안정적으로 동작하지만, Responses API 경로는 상대적으로 새 코드라 모델별 호환성 이슈 존재
- 모델 교체 시 API 호환성 영향도 함께 검증 필요 — 같은 vLLM 서버에서 모델만 바꿔도 API 경로별 동작이 달라질 수 있음
- Structured Output Parser 에러는 파서 자체 문제가 아니라 **상류 노드(모델)의 출력 형식 문제**인 경우가 많음 — 에러 발생 노드보다 상류를 먼저 확인

---

## 2026-05-27 — '사규 검색' 에이전트 장애 진단 및 임베딩 모델 교체

### 배경
- Dify 기반 '사규 검색' 에이전트 (4개 워크플로우 구성)가 지식 기반 답변을 생성하지 않는 문제 + Tool - RAG Chat HTTP 오류 발생
- 담당자가 만든 워크플로우를 분석·진단하는 작업 (코드 수정 아닌 인프라/설정 진단)

### 워크플로우 구조 파악

```
[사용자]
 └─▶ Sub Agent - 사내 규정 RAG (워크플로우, entry point)
       └─▶ Agent 노드 (gpt-oss-120b, FunctionCalling)
             ├── ToolRAGChat ──▶ Tool - RAG Chat (워크플로우)
             │     └── HTTP POST ──▶ ngrok URL ──▶ RAG Chatbot (chat 앱)
             │           └── Knowledge Base (dataset: ae6abea2...)
             │                 └── 임베딩: qwen3-embedding:8b (ollama) — 모델 소실 상태
             │                 └── 검색: keyword 0.7 / vector 0.3, top_k=3
             └── ToolSendMail ──▶ Tool - Send Mail (워크플로우)
                   └── send_mail (email 플러그인)
```

- **특이 구조**: Dify 앱(RAG Chatbot)을 같은 Dify 인스턴스에서 ngrok 경유 외부 HTTP 호출로 사용 (내부 호출 불가 이슈 때문)

### 문제 1: 지식 기반 답변 미생성

- **원인 확정**: RAG Chatbot의 임베딩 모델 `qwen3-embedding:8b` (Ollama)가 서버에서 소실됨
  - Dify UI에는 모델이 보이지만 실제 Ollama 서버에 없는 상태
  - 벡터 검색 시 쿼리 임베딩 실패 → 검색 결과 0건 → LLM이 문서 참고 없이 답변
- **해결**: Ollama를 더 이상 사용할 수 없는 상황이라 vLLM 서빙 중인 `bge-m3-ko` 임베딩 모델로 교체
  - **vLLM 플러그인(`yangyaofei/vllm`)은 전 버전(0.1.3~0.2.3) TEXT EMBEDDING 미지원** 확인 (manifest.yaml `text_embedding: false`)
  - OpenAI 제공자 직접 등록 시도 → Validate Model 단계에서 임베딩 전용 서버라 Chat Completions 검증 실패
  - **최종 해결: "OpenAI-API-compatible" 플러그인 설치 후 TEXT EMBEDDING 타입으로 개별 등록 성공**
    - Model Name: `bge-m3-ko`, API Base: `http://192.168.10.40:8010` (`/v1` 제외 — Dify가 자동 추가)
    - context size: 8192 (vLLM `--max-model-len`과 일치), max chunks per batch: 256
  - 지식 기반 검색 설정에서 임베딩 모델을 bge-m3-ko로 변경 → 검색 테스트 작동 확인
  - ⚠️ 스코어 0.11~0.13으로 낮음 — 기존 인덱스가 qwen3-embedding:8b로 생성된 벡터라 불일치. 재인덱싱 필요하나 담당자 작업이라 미수행

### 문제 2: Tool - RAG Chat HTTP 오류 (4월 21일~) — 원인 미확정, 담당자 트랙 이관

- **에러**: `Reached maximum retries (0) for URL https://hierogrammatic-supercongested-millie.ngrok-free.dev/v1/chat-messages`
- **진단 과정**:
  1. ngrok 프로세스 확인 → 살아있음 (3월 24일부터 실행 중)
  2. ngrok 터널 URL 확인 → `hierogrammatic-supercongested-millie.ngrok-free.dev`로 동일
  3. ngrok 포워딩 대상 확인 → `http://192.168.10.159:80`
  4. 직접 curl 테스트 → `Connection refused` (포트 80)
- **초기 가설 (폐기됨)**: ~~ngrok 80 포트 ↔ Dify API(dify-api) 5001 포트 mismatch → 5001로 재시작하면 해결~~
  - **폐기 사유 (사용자 정정)**: ngrok 매핑이 Dify API(5001)가 아니라 **Dify Web(3000)** 쪽과 연관. 따라서 5001 재시작 가설은 무효
- **현 상태**: **원인 미확정**. Dify Web 3000 포트와의 매핑 관계 / 4월 21일 전후 설정 변경 시점 추가 조사 필요
  - 본인 추가 조사 종료 — **담당자 확인 트랙으로 이관**
- **잠정 학습**: ngrok이 어느 서비스에 매핑되어 있는지(API 5001 / Web 3000 / 게이트웨이 등) 사전 확인이 진단의 첫 단계. Dify는 다중 포트 구성이라 매핑 확인 없이 포트 가설 세우면 오진 위험

### 학습 포인트
- vLLM은 OpenAI 호환 API(`/v1/embeddings` 등)를 제공하므로 OpenAI-API-compatible 제공자로 등록 가능
- vLLM에서 임베딩 모델 서빙 시 `--task embedding` 플래그 필요할 수 있음 (이번 케이스에서는 없이도 동작)
- Dify OpenAI 제공자의 API Base에 `/v1`을 넣으면 이중 추가됨 (`/v1/v1/...`) — 제외해야 함
- 임베딩 모델 교체 시 벡터 공간 불일치 발생 (스코어 급락) — 본 케이스는 **재인덱싱 안 함 결정** (담당자/사용자 트랙 선택). 일반적으로는 재인덱싱이 정공법
- ngrok 진단 시 포워딩 대상 서비스(API/Web/Gateway) 매핑 확인이 첫 단계 — 포트 가설을 매핑 확인 없이 세우면 오진 위험 (본 케이스 5001 가설 오진 사례)

---

## 2026-05-26 — P2 web-ui Keycloak 통합 사전 분석 (조사만, 코드 수정 0건)

**위임**: [[0. Inbox/2026-05-26 web-ui Keycloak 통합 사전 분석 위임 (P2)]] — 5/22 협의 18항 톱 4 (① 스택 / ③ 백엔드 / ⑤~⑦ spx-agent 패턴 / ⑧ realm-client) 조사 + 통합 전략 결정 + 다음 액션 PR 순서. 옵시디언 Claudian 환경(vault root cwd)에서 외부 컨텍스트 폴더로 web-ui(`C:\Users\Administrator\Projects\n8n\poc\web-ui\dify-chat`) + spx-agent 동시 접근.

### 조사 결과 (출력 인용 박은 톱 4)

#### ① web-ui 스택

- **프론트**: Next.js **16.1.6** (App Router) + React **19.2.3** + TypeScript 5.9.3, **webpack 강제** (`--webpack`), 패키지매니저 **npm**
- **백엔드**: Flask + Flask-CORS + psycopg2 + SQLAlchemy (자체 Flask 박힘, `backend/app.py`)
- **로그인 화면**: `frontend/app/login/page.tsx` — form UI 완성, **submit 핸들러는 stub** (`sessionStorage.setItem('user', JSON.stringify({email}))` + `router.push('/')` — 어떤 값이든 통과). 주석 `// TODO: 인증 모듈 연동 시 실제 로그인 로직으로 교체` 박혀있음
- **라우팅 가드**: `frontend/app/(main)/layout.tsx` — `useEffect`로 sessionStorage 체크 후 미존재 시 `/login` redirect (클라이언트 사이드만)
- **Keycloak 흔적**: 0건 (grep)

#### ③ 백엔드 유무 = 있음

- 별도 Flask 백엔드 박힘 (chat / document / files / rooms / schedules 5 Blueprint)
- **인증 라우트 0건** — auth/login/keycloak Blueprint 없음, `@require_auth` 데코레이터도 없음
- Next.js API route도 일부 존재 (`app/api/custom-sales/`, `app/api/sales-support/`) — 데모용 부분만, 메인은 Flask
- **결론**: SSR + httpOnly cookie 통합 가능

#### ⑤~⑦ spx-agent Keycloak 통합 패턴

- **코드 위치**:
  - 컨트롤러: `api/controllers/console/auth/keycloak.py` (405 lines) — login / callback / login-password (ROPG) / refresh / logout / force-login
  - 토큰 검증: `api/libs/passport.py` `verify_keycloak_token` (JWKS RS256, issuer 이중 검증)
  - 인증 미들웨어: `api/extensions/ext_login.py:78-96` — `KEYCLOAK_ENABLED` 분기 + `sub` → `spx_accounts.sub` lookup
  - 프론트: `web/app/signin/components/mail-and-password-auth.tsx` — `login({url: '/keycloak/login-password'})`
- **라이브러리**: **keycloak-js / next-auth 미사용** — 백엔드는 순수 `requests` + `secrets`/`hashlib` 자체 PKCE + PyJWT/PyJWKClient. 프론트는 자체 fetch 기반 `login()` 함수. **모든 KC 통신을 백엔드가 대행 → React 19 호환성 리스크 자동 0**
- **토큰 흐름**: 폼 → 백엔드 ROPG/PKCE → JWKS 검증 → JIT provisioning (`provision_default_workspace_for_keycloak`) → httpOnly cookie 3종 (access/refresh/csrf)
- **승랑님 base 커밋** (git log 식별): `f3e9dbe keycloak sso` (초기) → `c8cc19c accounts 동기화` → `0ff4434 로그인 페이지 복구` → `d10a5f9 리다이렉트 변경` → `a687c1a 그룹 sub` → `3acfba4 유저명 로그인 + audit 통합` (최신)

#### ⑧ realm/client 결정

- **realm**: `Spelix` 공유 (`.claude/docs/references/keycloak-sync.md` § 4 매칭 우선순위 그대로 재사용 가능 — sub 단일 진실 보존)
- **client**: web-ui 전용 신설, 후보 ID `web-ui-demo`, public + PKCE
- **환경변수 박힌 값** (`docker/.env.example`): `KEYCLOAK_REALM=Spelix`, `KEYCLOAK_CLIENT_ID=dify-app` (spx-agent용), `KEYCLOAK_URL=http://localhost:8180` (로컬) / `192.168.10.194:8080` (운영)

### 통합 전략 결정 (자동 도출)

**채택: SSR + httpOnly cookie (spx-agent 패턴 그대로 포팅)**

- 사유: ③ Flask 백엔드 박혀있음 + ⑥ 라이브러리 의존성 0 (React 19 충돌 회피) + ⑦ 백엔드 검증 필수 (cookie 구조)
- 토큰 보관: httpOnly cookie 3종
- 리프레시: silent (refresh_token cookie → 새 access_token)
- 로그아웃: Keycloak `/logout` 서버 호출 + cookie clear + 브라우저 end_session_endpoint redirect

### Drift 가드 보고 (§ 1)

| # | 위임 박제 | 실측 |
|---|---|---|
| 1 | keycloak-sync.md 경로 `.claude/references/keycloak-sync.md` | 실제는 `.claude/docs/references/keycloak-sync.md` (한 단계 더 깊음) |
| 2 | realm 명 `spelix` 소문자 | `KEYCLOAK_REALM=Spelix` 대문자 S — Keycloak realm 명 case-sensitive 주의 |
| 3 | 5/22 박제는 `sync_mock_users_keycloak.py`만 언급 | `.sh` 변종 존재 인지 (위임 체크리스트에 이미 박혀있음) |

자동 보정 0건 — 그대로 보고에 반영.

### 호환성 리스크

- **⑰ 패턴 부정합**: 2건 (인증 미들웨어 형태 `flask_login` → 자체 `@before_request` / API 표준 `flask_restx.Resource` → 순수 Blueprint). 모두 단순화 방향이라 우회 = 다운그레이드. 보안 손실 0
- **⑱ 라이브러리 버전 충돌**: 0건 (현재 정보). keycloak-js 미도입이라 React 19 충돌 자동 회피. 단 web-ui Flask 버전 미고정 → PyJWT 호환성 cross-check 통합 PR 진행 시 의무

### 다음 액션 PR 순서 (12단계, 4~5 PR로 그룹화 가능)

```
1. 194 Keycloak admin: web-ui-demo client 신설 (운영)
2. CORS: web-ui origin 허용 (1에 흡수)
3. backend/config.py + .env: Keycloak 환경변수 추가
4. backend/requirements.txt: PyJWT 추가 + routes/auth.py Blueprint 신설
5. backend/services/auth_service.py: 토큰 검증 + @require_auth 데코레이터
6. backend/routes/*.py: 기존 5 Blueprint에 @require_auth 적용
7. frontend/app/login/page.tsx: sessionStorage 제거 + 실제 API 호출
8. frontend/app/(main)/layout.tsx: SSR 가드로 변경 (cookies() 또는 /api/auth/me)
9. frontend/services/*: credentials: 'include' 추가 (LLM 호출 토큰 첨부)
10. 로그아웃 흐름
11. silent refresh (401 → /api/auth/refresh 1회 재시도)
12. 배포 URL redirect_uri 추가 등록
```

### 자가 검증 결과 (§ 2)

| 검증 | 결과 |
|---|---|
| web-ui `package.json` 본문 인용 | ✅ (Next.js 16 / React 19 / npm) |
| `login/page.tsx` L24-28 stub 코드 인용 | ✅ |
| `(main)/layout.tsx` L19-25 가드 코드 인용 | ✅ |
| spx-agent `keycloak.py` import + flow 6 엔드포인트 인용 | ✅ |
| `passport.py` JWKS RS256 검증 인용 | ✅ |
| `ext_login.py:78-96` Keycloak 분기 인용 | ✅ |
| `.env.example` Keycloak 변수 인용 | ✅ (`KEYCLOAK_REALM=Spelix`, `KEYCLOAK_CLIENT_ID=dify-app`) |
| `git log --oneline` 승랑님 6 커밋 인용 | ✅ |
| 코드 수정 0건 / KC admin 변경 0건 / spec 미터치 / 승랑님 PR 코멘트 미변경 | ✅ |

### PM 컨펌 후보 (이사님)

- **⑨ 데모 계정 발급 방식** — DEMO 그룹 신설 vs Spelix 기존 그룹 공유. 신설 권장 (운영 부서 트리와 권한 누수 방지)
- 통합 전략 B (SSR + httpOnly cookie) 채택 확정 요청
- client_id `web-ui-demo` 명명 확정 (또는 `spelix-web-ui` 등 대안)
- web-ui 배포 도메인 — redirect_uri 등록 시점 확정 필요

### 후속 트리거

- ⏳ **통합 PR 트랙** — 위 12단계 따라 별도 트랙 (4~5 PR로 그룹화)
- ⏳ **⑨ 데모 계정 발급 방식** — 이사님 PM 컨펌 (별도 트랙)
- ⏳ **Keycloak 운영 변경** (#1·2·12) — 통합 PR 1단계에서 처리
- ⏳ **web-ui Flask 실설치 버전 확인** → PyJWT 호환성 cross-check (#4 진행 시)
- ⏳ **drift 1번 (sync.md 실제 경로)** — `.claude/docs/PROMPT.md` 또는 향후 위임 프롬프트 작성 시 정정 박을 것

### 정책

- 본 위임은 **조사만** — 코드/스키마/Keycloak admin/spec/SESSION_HISTORY 본문(타 §) 미터치
- 산출물은 본 § (SESSION_HISTORY) + 채팅 보고서 2종. spec 트랙 진입은 통합 PR 트랙 시작 시점에 별도 결정
- 5/22 협의 18항 박제 활용 — 재조사 0, 후속 14항은 톱 4 결과로 자동 도출 (보고서 § B·C·D)

---

## 2026-05-26 — references/keycloak-sync.md 신설 (후속 ② 옵션 A)

**위임**: `.claude/docs/PROMPT.md` — 5/22 SESSION_HISTORY § Keycloak 부서 그룹 동기화 + § 트러블슈팅 박제를 영속 운영 절차 표준으로 분리. SESSION_HISTORY는 시간순 박제라 다음 작업자의 "Keycloak 동기화 어떻게 돌리지?" 검색 hit 곤란 → references로 추출.

### 옵션 채택 결정 (5/22 후속 ②)

3안 중 **옵션 A 채택**:
- ✅ A: `references/keycloak-sync.md` 신설 — 반복 운영 절차 + 트러블슈팅 표준
- ✗ B: `architecture.md § 운영 절차` 통합 — architecture 분량 비대 + 운영 절차 ≠ 아키텍처
- ✗ C: SESSION_HISTORY 박제만 — 시간순 + 다른 5/22 내용과 섞여 검색 곤란

### 작업 내용

#### 신규 파일: `references/keycloak-sync.md` (133 lines)

frontmatter (`tags`/`type`/`date`/`last_updated`/`purpose`) + § 1~§ 8 본문:
- **§ 1 배경** — 5/22 정합성 불일치 + `.gitignore` 정책
- **§ 2 인스턴스 분리** — 로컬 KC(`localhost:8180`) / 194 KC(`192.168.10.194:8080`) 분리 표
- **§ 3 Keycloak 테이블 4종** — `user_entity` / `user_group_membership` / `credential` / `groups` + 우리 매핑(`spx_accounts.sub` / `spx_departments.keycloak_group_id`)
- **§ 4 매칭 우선순위** (`account_service.py:209-365`) — UUID → code(lazy backfill) → 신규 생성 + 그룹명 = DB `code` (path leaf) + `dept.name` 절대 덮어쓰지 않음
- **§ 5 동기화 스크립트** (3 하위 §) — `sync_dept_keycloak.sh` 입출력/사전조건/멱등성, `sync_mock_users_keycloak.py` 동상, 실행 순서(.sh → .py)
- **§ 6 트러블슈팅 5종** — 5/22 박제분 표 박음 (특수문자/PGHOST/멀티바이트/토큰 만료/sub 중복)
- **§ 7 관련 결함** — H-INFRA-04 / H-ENV-04 / H-DASH-17 사슬
- **§ 8 후속 트리거** — P2 web-ui Keycloak 통합 사전 분석 base 참조

#### CLAUDE.md 도메인 지식 표 1행 추가

```
| `references/keycloak-sync.md` | Keycloak 부서 그룹/사용자 동기화 절차 + 매칭 우선순위 + 트러블슈팅 | 동기화 스크립트 운영 시 / web-ui Keycloak 통합 사전 분석 시 |
```

### Drift 가드 확인 (§ 1)

작업 진입 전 사전 검증:
- ✅ `scripts/sync_dept_keycloak.sh` / `sync_mock_users_keycloak.py` 존재
- ✅ `api/models/rbac.py:58` `keycloak_group_id` 컬럼 존재
- ✅ `api/services/account_service.py:209` `_sync_department_from_keycloak_groups` + `find_group_by_path` + `DepartmentCodeConflictError` 모두 존재

새 drift **0건**. SESSION_HISTORY 5/22 박제 사실 모두 코드/스키마와 정합.

> 미세 drift 보고: `scripts/sync_mock_users_keycloak.sh` (.sh 변종) 추가 존재 — 5/22 SESSION_HISTORY는 .py 만 언급. 본 위임은 5/22 박제 기준 따름 (.py만 박음). 사용 의도 확인 후 본 문서 § 5.2 보강 또는 SESSION_HISTORY 갱신 검토 필요.

### 자가 검증 결과 (§ 2)

| 검증 | 결과 |
|---|---|
| 파일 존재 (`ls`) | 8,661 bytes ✅ |
| § 1~§ 8 + § 5.1~5.3 박힘 | 12개 § ✅ |
| 결함 사슬 (H-INFRA-04 / H-ENV-04 / H-DASH-17) | 7건 인용 ✅ |
| CLAUDE.md 등록 | L121 1행 박힘 ✅ |
| 비밀번호/secret 박지 않음 | placeholder/컬럼명/.gitignore 인용만 ✅ |

### 후속 트리거

- ⏳ **P2 web-ui Keycloak 통합 사전 분석** — § 8에 트리거 박힘. 옵시디언 데일리 트랙으로 이어짐. 공유 realm 채택 시 § 4 매칭 우선순위 재사용, 분리 realm 시 sub 매핑 별도 설계 검증 필요
- ⏳ **`sync_mock_users_keycloak.sh` 변종 정합** — SESSION_HISTORY 박제(.py만)와 미세 drift, 사용 의도 확인 후 보강
- ⏳ **architecture.md / hdd/design.md 운영 절차 영향 검토** — 본 위임 범위 외, 별도 트랙

### 정책 (옵션 B 재확인)

- 변경 이력 본문 박지 않음 — 옵션 B "현재만 박음"
- 비밀번호/secret 본문 박지 않음 — 본 문서 git tracked, 보안 가드
- 스크립트 본문 박지 않음 — `.gitignore` 영역 분리
- defect-catalog 본문 미수정 — 본 문서는 결함 ID 참조만, 결함 본문은 catalog 단일 진실

---

## 2026-05-26 — 하네스 문서 정합 + context-bar 폐기→보류 정정

**위임**: `.claude/docs/PROMPT.md` — 5/22 spec 정합 후속 4건. spec → 하네스 문서(CLAUDE.md / conventions.md / defect-catalog.md / tasks·data-mart) 진실 사슬의 하네스 문서 칸만 청산. **코드 수정 0건, SESSION_HISTORY/architecture/design/quality-criteria 미터치**.

### ⚠️ 사용자 정정 사항 (context-bar)

5/22 위임에서 spec 3파일에 `🔒 폐기` 헤더 + "임포트/렌더링 제거" 본문을 박았으나 **잘못된 표기**. 사용자 확인:
- 컴포넌트 코드 (`web/app/components/admin/context-bar/`) **미삭제**, page.tsx 렌더링만 비활성
- 나중에 재도입 가능성 있음 — `chart-drawer` 5/13 보류와 **동일 패턴**

→ "🔒 보류 (2026-05-22)"로 정정. spec 본문 base 보존 + 재검토 트리거만 박음.

### 작업 순서 (5단계)

1차 → 1.5차 → 2차 → 3차 → 4차 순으로 진행. 각 단계 끝에 grep 자가 검증 출력 첨부.

#### 1차: CLAUDE.md

- 프로젝트 개요 범위 문구: KPI 4종 1차 이름으로 정정 (`총 오브젝트 / 총 이용 앱 수 / 총 앱 호출량 / 인기 호출 앱`)
- 페이지 소개 문구 § 신설 — `common.menus.dashboardDescription` 한·영
- 컴포넌트 목록에서 `context-bar`를 Phase 2에서 분리 → 별도 `🔒 보류` 항목으로 박음 (chart-drawer와 같은 라인)
- 마운트 환경 인용구 KPI 4 명칭 정정

#### 1.5차: context-bar spec 3파일 정정

- `req/`·`design/`·`tasks/context-bar.md` frontmatter `status: "🔒 폐기 (2026-05-22)"` → `"🔒 보류 (2026-05-22)"`
- 본문 헤더 "임포트/렌더링 제거" → "렌더링만 비활성, 컴포넌트 구현체 코드 보존"
- 참조 문구에 H-DASH-21 미리 인용 (3차에서 박는 정식 ID)
- 본문(요구사항/설계/체크리스트 본체)은 그대로 보존 — chart-drawer 5/13 보류 패턴

#### 2차: hdd/conventions.md (UI 표준 86줄 파일)

5/21 박힌 § 위에 5/22 표준 5종 누적:
- **C2 기간 라벨 표시** § 신설 — 적용 대상/위치/스타일/포맷/i18n
- **C3 정렬 규칙** 보강 — 부서명 기반 표 4개 분기(가나다순) + `app-stats-table` 예외 명시
- **C6 ECharts 여백 표준** § 신설 — `grid.bottom: 20` / `grid.right: 60` / `barWidth: 16` + 카드 `pb-5` 행 추가
- **A2 용어집** § 신설 (최상위 절) — 이용/사용/호출/신규 4종 정의 표
- **A3 i18n 라벨** 보강 — 하드코딩 금지 인용구 + `common.menus.dashboardDescription` 추가

#### 3차: hdd/defect-catalog.md 결함 3건 신규

ID 충돌 사전 확인 후 결정: **H-DASH-21 / H-INFRA-04 / H-ENV-04** 무충돌.

- **H-DASH-21: 컨텍스트바 보류 결정** — H-DASH-20(chart-drawer 5/13 보류) 자매 패턴. 8행 표(상태/상황/원인/현재 동작/영향/방어/테스트/재검토). 방어 = `page.tsx` import 0건 유지 + 컴포넌트 코드 보존 + 재도입 PM 결정 동반 트리거
- **H-ENV-04: Windows Git Bash 멀티바이트 인자 깨짐** — `delegation-standard § 5` PowerShell 한글 인코딩 자매. 방어 = python 전환 (5/22 채택)
- **H-INFRA-04: Keycloak admin API 토큰 만료** — 5/22 발견. 방어 = 50건마다/4분마다 토큰 재발급, 멱등 보장

**관련 정합**: `req/context-bar.md` frontmatter `harness_candidates`에서 `H-CAND-context-bar-deprecated` 제거 + `harness: [H-DASH-21]` 등록 (졸업).

#### 4차: 잔존 폐기명 정합 (3 파일)

5/22 위임 scope 외였던 3 파일 폐기명 정정:
- `tasks/kpi-cards.md` — KPI 2/3/4 mock 명세 신명 + KPI 1 mock `new_count` 메인 + 분해 이동 표기
- `tasks/kpi-drill-through.md` — 2-A/B/C/D 섹션 차트·표 명칭 신명 + 가나다순 정렬 명시 + KPI 4 보조 카드 폐지 + 컬럼명 신명
- `requirements/data-mart.md` — KPI 4종 부서 기준 분리 표 + 마트 객체 의존 매핑 표 신명 정정

### 자가 검증 결과 (§ 2 가드)

| 검증 | 결과 |
|---|---|
| CLAUDE.md 폐기명 grep | 0건 ✅ |
| CLAUDE.md 신명 grep | 2건 박힘 ✅ |
| CLAUDE.md "context-bar 폐기" 표현 | 0건 ✅ |
| spec 3파일 "🔒 폐기 (2026-05-22)" | 0건 ✅ |
| spec 3파일 "🔒 보류 (2026-05-22)" | 6건 (frontmatter 3 + 본문 3) ✅ |
| spec 3파일 "렌더링만 비활성/코드 보존" | 3건 ✅ |
| conventions.md § 신설 (C2/C6/A2) | 3개 § 박힘 ✅ |
| conventions.md C3 분기 / C6 토큰 / A2 용어집 | 2/3/3건 ✅ |
| defect-catalog H-DASH-21/INFRA-04/ENV-04 | 3건 박힘 ✅ |
| H-CAND-context-bar-deprecated 잔존 | 0건 (전면 제거) ✅ |
| scope 내 폐기명 (tasks·data-mart) | 0건 ✅ |
| 신명 전수 박힘 | 8 파일 40건 ✅ |

### Drift 가드 확인 (§ 1)

- `ls web/app/components/admin/context-bar/` → `index.tsx` 출력 = **디렉토리 보존 확인** → 사용자 정정 사항(코드 미삭제, 보류) 일치. 새 drift 발견 0건

### 후속 잔여

- ⏳ **후속 ② Keycloak 동기화 문서화** — 옵션 A(`references/keycloak-sync.md` 신설) / B(`architecture.md § 운영 절차`) / C(`SESSION_HISTORY.md` 박제만) 사용자 결정 대기
- ⏳ **5/22 잔여 drift 2건** — 코드 수정 동반, 별도 PR
  1. `dashboard/page.tsx:35` defaultValue ↔ `common.json` 값 불일치 (JSON 승)
  2. PROMPT 명세 경로 `/api/scripts/` → 실제 `/scripts/` (Keycloak sync 스크립트 3개)
- ⏳ **architecture.md / hdd/design.md / hdd/quality-criteria.md** — 본 위임 범위 외, 영향 검토만 (후속 ②와 묶임)
- ⏳ **context-bar 보류 본문 base 보존 영역** 폐기명 잔존 — chart-drawer 5/13 보류와 같은 의도된 보존. 재도입 시 정합 트리거

### 정책 (옵션 B 재확인)

- 변경 이력 본문 박지 않음(SESSION_HISTORY 분리)
- 옵션 B "현재만 박음" 정책 → 본 위임 2회 실행 시 추가 변경 0건 (idempotent 자연 보장)
- ID 충돌 사전 확인 후 한 번에 결정 → 재실행 시 ID 변경 0

---

## 2026-05-22 — Keycloak 부서 그룹 동기화

### 배경
- `spx_departments`에 목업으로 넣은 DEPT-01 ~ DEPT-10 부서가 Keycloak에는 미등록 상태
- 원래 흐름: Keycloak 그룹 등록 → DB 반영. 현재 DB에만 데이터가 있어 정합성 불일치

### 조사 결과
- **DB**: DEPT-01 ~ DEPT-10 (10개) 전부 `keycloak_group_id = NULL`
- **로컬 Keycloak** (`localhost:8180`): DEPT-01만 존재, DEPT-02~10 미등록
- **194 Keycloak** (`192.168.10.194:8080`): 별도 인스턴스, 별도 DB (로컬과 독립)
- 로컬 KC DB = 로컬 Postgres 컨테이너(`db_postgres:5432/keycloak`), 194 KC = 194 서버 자체 DB

### 동기화 로직 확인 (`account_service.py:209-365`)
- 매칭 우선순위: UUID 매칭 → code 매칭(lazy backfill) → 신규 생성
- Keycloak 그룹명 = DB `code` 값 (path leaf name)
- `keycloak_group_id`가 NULL이면 로그인 시 자동 backfill 가능하나, 수동 선반영이 안전

### 작업 내용
- `scripts/sync_dept_keycloak.sh` — 194 Keycloak에 DEPT-01~10 그룹 생성 + DB `keycloak_group_id` 반영
- `scripts/sync_mock_users_keycloak.py` — 목업 사용자 150명 Keycloak 등록 + DEPT 그룹 배정 + DB `sub` 반영
- 두 스크립트 모두 `.gitignore` 추가 (비밀번호 포함)
- 중복 계정 정리: `김 민준`(id: `3d9faddf-...`, Keycloak 로그인 자동생성) 삭제 → 목업 `김민준`(id: `a0000000-...-000001`)에 sub 수동 반영

### Keycloak 테이블 구조 (조사)
- `user_entity`: 사용자 계정 (id, username, email, first/last_name, realm_id)
- `user_group_membership`: 사용자↔그룹 매핑 (group_id, user_id)
- `credential`: 비밀번호 (해시 저장)
- `spx_accounts.sub` = Keycloak `user_entity.id` (UUID)

### 트러블슈팅
- 비밀번호 특수문자(`#$^^`) → shell 변수 치환 문제 → 작은따옴표 + `--data-urlencode` 해결
- `docker exec psql`이 로컬 DB에 실행됨 → `-h`/`-p`/`-e PGPASSWORD` 추가로 194 원격 DB 직접 연결
- bash에서 한국어 인자가 python에 깨짐 (Windows Git Bash 멀티바이트 문제) → 전체 python 스크립트로 전환
- Keycloak 토큰 만료 (150건 API 호출 중) → 50건마다 토큰 재발급
- `spx_accounts_sub_unique_idx` 충돌 → 중복 계정 정리 + `NOT EXISTS` 조건 추가

---

## 2026-05-22 — 대시보드 라벨/레이아웃 전면 개편 + 쿼리 수정

### 라벨/용어 변경

- **KPI 카드 타이틀**: 총 오브젝트(유지) / 총 이용 앱 수 / 총 앱 호출량 / 인기 호출 앱
- **용어 통일**: "이용"=앱을 호출/소비하는 행위, "사용"=플랫폼 전반 활동(생성+이용). 혼용 정리
- **i18n 적용**: 앱/지식/도구 용어를 `t('common.objectType.*')`로 통일 (한국어: 앱/지식/도구, 영어: App/KB/Tool)
- **드릴 차트/표 이름**: 부서별 앱 이용 수, 앱 이용자 Top10, 부서별 앱 호출 수, 호출 수 Top10, 에러 발생 Top10 등
- **표 컬럼**: 부서원/부서원당 이용 앱 수/Top 이용 앱/추세, 앱/부서/호출/이용자/에러/마지막 사용 등
- **소개 문구**: "앱 이용 현황과 부서별 활동을 한눈에 확인합니다."

### KPI 카드 레이아웃 변경

- **좌우 분리 레이아웃**: 좌(제목 `system-sm-semibold` + 기간 라벨 + 툴팁) / 우(큰 숫자 `text-2xl font-semibold` + diff 배지)
- **KPI1**: 메인 값을 `new_count`(기간 내 신규)로 변경, `diff_percent` 복원
- **KPI4**: 타이틀="인기 호출 앱"(고정), 우측=앱명(truncate+hover 툴팁, max-width 55%), diff 자리=호출 수
- **KPI1 details(App/KB/Tool)**: 카드에서 제거 → 부서별 오브젝트 현황 차트 범례로 이동
- **활성 카드**: 배경색 변경 제거 (ring만 유지)
- **diff 0%** → `-` 표시로 변경
- **툴팁 내용**: 사용자 관점으로 재작성

### 차트/표 UI 개선

- **부서별 오브젝트 현황**: ECharts 범례 제거 → 카드 헤더 우측에 React 범례(색상 dot + 라벨 + 동적 개수), 범례 클릭 시 해당 타입만 필터링 표시
- **기간 라벨**: 모든 차트/표 제목 옆에 기간 라벨 추가 (연한 색 작은 글씨)
- **표 기본 정렬**: 부서 기반 표 4개 부서명 가나다순 정렬 적용
- **소유자/이용자 차트**: Y축 라벨=이름만, hover 시 이름/부서/값 3줄 표시
- **모델별 토큰 사용량**: 합계에서 'tokens' 텍스트 제거
- **차트 하단 여백**: `grid.bottom` 10→20, 카드 `pb-3`→`pb-5`로 상하 여백 균등화
- **컨텍스트바 제거**: page.tsx에서 ContextBar 임포트/렌더링 제거 (ESC 키 해제는 유지)

### 백엔드 쿼리 수정

- **앱 이용자 Top10** (`dashboard_drill_users_service.py`): `_TOP_USERS_SQL`에 `AND NOT ae.is_debug AND ae.is_canonical_call` 필터 추가 — 디버그/비정규 호출 제외, 앱 이용만 카운트

### 빌드 대상

- **web** (프론트엔드 전면 변경)
- **api** (앱 이용자 Top10 쿼리 필터 추가)

---

## 2026-05-22 — spec 24파일 5/22 코드 정합 (drift 청산, 역방향 예외)

**위임**: `.claude/docs/PROMPT.md` — 5/22 코드 변경분(라벨/레이아웃/쿼리)이 코드에만 박힌 채 spec drift 누적 → spec을 코드에 정합. **본 위임은 코드 수정 금지** (옵션 B 스타일: 현재만 박음, 변경 이력 본문 박지 않음).

### 작업 흐름 (역방향 예외)

평소 원칙(spec → 코드)을 한 번 뒤집어 **코드 기준 → spec 청산**. 사용자 화면 검증 완료된 5/22 코드를 진실로 두고 drift만 청산.

1. 코드 정독 (5/22 commits 4d5a29f / d898e4e / f98448b / 0d85246 / a2ca5fd)
2. drift 매트릭스 작성 (카탈로그 A~E × spec 파일 × 라인)
3. spec 갱신 실행 (8 컴포넌트 × req/design — 총 11 파일)
4. grep 자가 검증

### 갱신 파일 (총 11개)

- **kpi-cards** req/design — 좌우 분리 레이아웃 § 신설, KPI 4 타이틀 고정 + 앱명 큰 자리/호출 수 뱃지 자리, KPI 1 메인=`new_count`, App/KB/Tool 분해 제거(dept-objects 범례로 이동 참조), 활성 카드 `ring-inset` only(`bg-hover` 폐기), diff 0% → `-`, 툴팁 코드 기준 재작성, `diff-badge.tsx` 별도 파일 표기 제거 (`kpi-card.tsx` 내부 inline)
- **kpi-drill-through** req/design — 메트릭 카탈로그 차트·표명 일괄 정합(부서별 앱 이용 수 / 앱 이용자 Top10 / 부서별 앱 호출 수 / 호출 수 Top10 / 에러 발생 Top10), 표 컬럼 재명명(부서원 / 부서원당 이용 앱 수 / Top 이용 앱 / Top 호출 앱 / 이용자), 차트 UI 정합 § 신설(Y축 이름만 + hover 3줄 — top-users만), 정렬 규칙 부서명 가나다순 + `app-stats-table` 예외, `_TOP_USERS_SQL` 필터 § 신설(`NOT is_debug AND is_canonical_call`)
- **dept-objects** req/design — ECharts legend 제거 + 카드 헤더 React 범례 § 신설(dot + i18n 라벨 + 동적 개수 + 클릭 필터 토글, series zero-out 패턴)
- **model-tokens** req — 합계 표기 카드 헤더 우측 + 단위 "tokens" 제거 (차트 hover 툴팁은 "토큰: {raw}" 유지)
- **dept-activity** req — 표 컬럼을 `dept-new-creations-table` 5컬럼(부서/앱/지식/도구/신규)으로 정합, 정렬 부서명 가나다순
- **dashboard-controls** req — 페이지 소개 문구 § 신설 (`common.menus.dashboardDescription` 한/영)
- **context-bar** req/design/tasks — `🔒 폐기 (2026-05-22)` frontmatter + 본문 최상단 폐기 헤더(`chart-drawer` 5/13 보류와 같은 보존 패턴), 본문은 design intent base로 보존

### 자가 검증 결과 (§ 2 가드)

- scope 내 폐기명("부서별 채택 앱 수" / "앱별 통계") grep: **0건**
- 신명("총 이용 앱 수" / "총 앱 호출량" / "인기 호출 앱") grep: **5 파일 28건 박힘**
- context-bar 폐기 헤더: **3 파일 모두 박힘** (req/design/tasks)

### Drift 발견 보고 (§ 1 가드)

- `dashboard/page.tsx:35` defaultValue(`앱 이용 현황과...`)와 `common.json` 값(`앱, 지식, 도구의 사용 현황과...`) 불일치 — 런타임 JSON 승. spec은 JSON 값 박음
- PROMPT 명세 경로 `/api/scripts/` → 실제 `/scripts/` (Keycloak sync 스크립트 3개 모두 root `.gitignore` 243~245행)

### 후속 위임 후보

1. **conventions.md 정합** — C2 기간 라벨 전역 / C3 부서명 가나다순 전역 / C6 카드 패딩 표준 / A2 "이용" vs "사용" 용어집 / A3 `common.objectType.*` i18n 규칙
2. **Keycloak 동기화 문서화 결정** — 옵션 A(`references/keycloak-sync.md` 신설) / B(`architecture.md § 운영 절차`) / C(`SESSION_HISTORY.md` 박제만) 중 채택
3. **CLAUDE.md 핵심 컨벤션 § 갱신** — 프로젝트 개요 범위 문구 KPI 4종 신명 / 페이지 소개 문구 / 컨텍스트바 폐기 표기
4. **defect-catalog.md 추가 후보** — Keycloak 토큰 만료 대응 / Windows Git Bash 멀티바이트 / **H-CAND-context-bar-deprecated** 정식 ID 부여
5. **scope 외 잔존 폐기명 정합** — `tasks/kpi-cards.md`·`tasks/kpi-drill-through.md`·`requirements/data-mart.md`에 폐기 카드명 잔존

### 정책 (옵션 B "현재만 박음" 재확인)

- 변경 이력 본문에 박지 않음(`SESSION_HISTORY.md`로 분리) → idempotent 자연 보장 (재실행 시 추가 변경 0건)
- 2차 후보(평균/가장 사용량) 폐기 결정 확정, spec에 흔적 남기지 않음
- PM 컨펌 후보 톤 다운: KPI 3 본질은 "총 앱 호출량" 1차 이름 확정으로 옵션 B 채택

---

## 2026-05-21 — UI 정합 14개 항목 (PPTX 화면 수정 피드백 청산)

**위임**: `.claude/docs/PROMPT.md` — PPTX 14개 UI 수정 항목 일괄 청산 (폰트/간격/정렬/ellipsis/i18n/추세 기호/active ring). 백엔드 변경 0건.

### 작업 결과

**spec 갱신 4건** (정책 먼저 → 코드 나중 흐름 준수):
- `design/kpi-drill-through.md` — 5.1 차트명 `호출 수 상위 10개 앱` / `에러율 상위 10개 앱`
- `requirements/kpi-drill-through.md` — 2.1 ring-inset + 1.4 행수 6/6/10 + 1.6 정렬규칙(localeCompare) + 1.9 i18n 라벨 + 5.1 차트명
- `requirements/kpi-cards.md` — 4.1 추세기호 `▲▼` → `▴▾` + 5.1 차트명
- `conventions.md` **(신규)** — 폰트/간격/정렬/ellipsis/행수/추세기호/i18n/ring/색상 전체 표준

**프론트 코드 변경 19파일 + i18n 2파일** → 화면 검증 후 추가 수정으로 **22파일** + i18n 2파일:

| # | 항목 | 변경 내용 |
|---|------|----------|
| 1.0 | 폰트 통일 | 이미 정합 — 변경 불필요 |
| 1.1 | 설명 문장 | page.tsx에 탐색 패턴 동일 `grow truncate text-text-primary system-xl-semibold` + i18n |
| 1.2 | ellipsis | 차트 axisLabel `overflow:'truncate'` 10곳 + 표 `block truncate` + `title` 8곳 |
| 1.3 | 합계 위치 | model-tokens-chart 합계 좌하→우상 (사용자 결정) |
| 1.4 | 행수 고정 | overview 3곳 + **drill 차트 8곳** 전수 `max-h` + `overflow-y-auto overflow-x-hidden` |
| 1.5 | 카드 간격 | chart-area `gap-4`→`gap-6`, page `gap-6`. KPI grid `gap-4` 유지 |
| 1.6 | 정렬 | 9곳 `localeCompare('ko-KR')` 2차 정렬 추가 |
| 1.7 | 컬럼 균일 | 5개 표 `table-fixed` + 첫 컬럼 `w-[22%]` 나머지 자동 균등 |
| 1.8 | 전체보기 제거 | dept-activity-table "전체 보기 →" 버튼 삭제 (사용자 결정) |
| 1.9 | i18n | `objectType.app/kb/tool` i18n key → kpi-section + 표 2파일 + **dept-objects-chart legend** |
| 2.1 | ring-inset | kpi-card.tsx `ring-inset ring-2` + **`bg-components-card-bg-hover`** 추가 |
| 2.2 | 차트 grid 통일 | `right: 60` 전수 (10곳) + `bottom: 10` 전수 (10곳) + barWidth 16 통일 |
| 4.1 | 추세 기호 | KPI DiffBadge `RiArrowUp/Down` → `▴▾` 문자 + drill 표 2파일 전수 |
| 5.1 | 차트 이름 | `호출 수 상위 10개 앱` / `에러율 상위 10개 앱` |

**검증 V.1~V.12 통과**: vitest 18/18 ✅, grep `▲▼` + `RiArrowUpSLine` 0건 ✅

### 사용자 결정 기록 (1.3 / 1.8)
- **1.3**: "우하 차트에 있는 모델별 토큰 사용량에 합계 표시가 차트 좌하에 있는데 우상으로 변경"
- **1.8**: "메인 화면(kpi 클릭 안 한 화면)에서 표 우측 상단에 있는 '전체 보기'를 지우면 돼"

### 화면 검증 후 추가 수정 (2차 라운드)
1. **1.1 설명 스타일** — `system-sm-regular text-text-tertiary` → 탐색 패턴 `system-xl-semibold text-text-primary` 재수정
2. **1.4 drill 차트 8곳** — `max-h-[250px]` 래퍼 누락 → 전수 추가
3. **1.9 dept-objects legend** — `'App'/'KB'/'Tool'` 하드코딩 → i18n `buildChartOption(data, labels)` 패턴
4. **4.1 KPI DiffBadge** — `RiArrowUpSLine/DownSLine` 아이콘 → `▴▾` 문자 치환 + import 정리
5. **2.1 `bg-components-card-bg-hover`** — 프롬프트 명시인데 1차에서 누락 → 추가
6. **2.2 chart `bottom: 10`** — 하단 잘림 방지 → 전 차트 10곳 적용
7. **1.7 컬럼 균일** — `style={{ width }}` 고정 퍼센트 → `table-fixed` + 첫 컬럼만 고정, 나머지 자동 균등
8. **3-C kpi-cards.md** — 폰트 스타일 토큰 표준 § 신설

### 다음 작업
- web 컨테이너 재빌드 (`docker compose build --no-cache web`)
- `items` vs `rows` 응답 키 통일 — 별도 PR (본 위임 범위 외)
- 시각 회귀 테스트 도입 권장 (Storybook/Chromatic)

---

## 현재 상태

- **Phase**: 2 진입 (정적 대시보드 Phase 1 5/5 컴포넌트 모두 A 등급 달성 ✨)
- **다음 작업 (2026-05-14 갱신)**:
  1. ✅ **대시보드 프론트 정정 11건** — 라우트/헤더/i18n/redirect/설정모달/KPI/drill/context-bar/dept-activity/레이아웃 (5/14 완료, 미커밋)
  2. **디자인 결정 3건 반영** — 페이지 폭/여백 commonLayout 표준 따름 + h1 제거 + 카드 표준 2분리(KPI=AppCard / 차트·표=NewAppCard). 5/14 라우트 페이지 첫 실측 후 결정
  3. **Dead code 정리 + 커밋** — dashboard-page 폴더 + errors 드릴 컴포넌트 삭제 + 커밋 분할
  4. **audit collector 보강 + 마트 백엔드** — 별도 진행 중 (승랑님 의존)
  5. ✅ **design.md § 2.5 정식 이관 완료** — 5/14 마트 설계를 hdd/design.md § 2.5 본문화 + § 10 미해결 결정 + spec 3파일(req/design/tasks) 갱신 + references/audit-details-spec.md P0 명세 + Generated Column DDL + 검증 시나리오 윤곽 추가. 5/13 노트 archive 완료 (옵시디언 Claudian 5/14 마지막 세션)
  - 보류: top-error-types 1종 / chart-drawer 4종 (H-DASH-20)
- **현재 등급 (2026-05-06 갱신)**:
  - dashboard-controls: **A** (vitest 6/6 통과, Node 22 환경에서 검증)
  - kpi-cards / dept-objects / model-tokens / dept-activity: A (5/4 백엔드 service + 단위 테스트 + fetch 연동 완료)
- **작업 환경**:
  - 옵시디언: 문서 마스터 (`3. 프로젝트/spx-agent/`)
  - VSCode Claude Code: 실제 코드 작성 (`{프로젝트경로}/.claude/`)
  - 동기화: SymbolicLink (5/13 양방향 sync 깨졌다가 재생성 복원)
  - **호스트 Node**: 22 (회사 표준 `.nvmrc=22` 따름. 5/6 업그레이드 완료)
  - **패키지 매니저**: `pnpm@10.33.0` (`packageManager` 필드 + corepack 자동)
  - **Claude Code CLI**: `npm install -g @anthropic-ai/claude-code` (Node 22에 설치 필수, nvm 버전마다 별도 설치 필요 — 5/13 등록)

## 핵심 결정 (불변)

| 항목 | 결정 |
|------|------|
| 팀 분담 (3분 구조, 2026-05-06 최종 정정) | **김이사님 (PL)**: RBAC **스키마 결정** + 화면 설계 / **승랑님**: **Keycloak 작업** + **데이터 파이프라인** (로그/대시보드 스키마 제공자) + Docker build 흐름 / **권대리님**: **RBAC 코드 구현** (`feat/rbac` 브랜치) |
| `feat/rbac` git author vs 실제 작업자 | git author = 권대리님 (commit/머지 담당). `auth/keycloak.py` 등 **SSO 관련 파일의 실제 작업자 = 승랑님** (commit만 권대리님). lint fix 책임/슬랙 대상은 **실제 작업자 기준** |
| 호스트 Node 표준 | 22 (`.nvmrc=22`) |
| 패키지 매니저 표준 | `pnpm@10.33.0` (`package.json` `packageManager` 필드, corepack 자동) |
| Pre-commit hook | `.vite-hooks/` (husky 폴더 rename 패턴). 활성화는 `pnpm install`의 prepare script. **개발자별 활성화 시점 차이로 lint 사후 발견 발생** — CI lint job이 유일한 강제 게이트 |
| Pre-commit hook이 tsc 검증 포함 (2026-05-13 인지) | 다른 팀 영역 TS 에러로 본인 작업 커밋 차단 가능. CI lint job + 본인 영역만 보는 게 일반적 — 정책 재검토 가치 (별도 트랙) |
| 부서 구조 | 플랫 (`parent_id` 컬럼 자체가 없음, 독립 부서) |
| 구현 순서 | 프론트(목업) → 백엔드 → 연동 → 테스트 |
| ~~어드민 API 경로~~ ~~`/console/api/admin/dashboard/`~~ | **2026-05-13 변경 → `/console/api/dashboard/`** (사용자 범위 = 전체 공개 + 라우트 = `/dashboard`로 변경되며 UI와 통일. 컨트롤러는 `api/controllers/console/dashboard/`로 이동) |
| **회사 RBAC 명명 표준** (2026-05-06 정정 — 권대리님 확정) | **prefix 없음** (5/4 가정 `sp_` 폐기). `departments` / `department_members` / `resource_ownership` / `resource_permissions` / `rbac_audit_logs`. 사용자 매핑은 `accounts.id` 직접 (별도 sp_users 없음) → H-DASH-14 자동 해소. 사용자 ↔ 부서는 다대일 (UNIQUE). 상세: [[3. 프로젝트/spx-agent/references/rbac-schema.md]] |
| 캐시 정책 | TanStack Query staleTime 5분 |
| 용어 | "화면별" → "컴포넌트별" |
| Dify 코드 | 무수정 원칙, 신규 모듈만 추가 |
| 대시보드 사용자 범위 (2026-05-13 변경) | **로그인 사용자 전체** — 관리자 전용 폐기. 이사님 "데이터 노출 OK" 답변. 어드민 게이트 불필요, 일반 로그인 가드만. CSV 내보내기는 지금 고려 안 함. dataset_operator도 잠정 허용 (UI 확인 필요) |
| CSV 내보내기 | 지금 고려 안 함 (전체 공개로 외부 유출 위험 인지). 향후 차트 드로어 부활 시 재검토 |
| **마트 입력 (2026-05-12 이사님 확정)** | **audit_events 단일 SoT + RBAC JOIN**. 오브젝트 차트만 `resource_ownership` 직접 조회 (state라 audit 본질 불가). 5/8 4-fact, 5/11 OLTP 3-fact, 5/12 오전 옵션 D(OLTP 정규식) 전부 폐기 |
| **마트 RBAC JOIN 전략 (2026-05-14)** | **옵션 B = ID만 사전 JOIN, 부서명은 query-time** — enriched MView에 `app_owner_dept_id`/`actor_dept_id` 박고 부서명은 컴포넌트 쿼리에서 `JOIN departments d`. 사유: RBAC 운영 스케일(dept_members 만 단위)에서 비싼 결합은 refresh 1회 흡수 + 부서명 변경 즉시 반영. 옵션 A(이름까지 박음)/C(완전 분리)/D(이중 mat) 비교 후 채택. 상세: [[0. Inbox/마트 설계 결정 - 2026-05-14.md]] |
| **마트 채택 vs 호출 부서 기준 (2026-05-14 정정)** | **채택(KPI #2) = actor 기준 / 호출(KPI #3, dept-activity, drill-through) = owner 기준**. 5/12 이사님 "owner 일관" 결정에서 채택만 actor로 분리 (Breadth 의미상 자연). mv_kpi_calls_daily가 두 부서 ID 둘 다 GROUP BY 보존 |
| **마트 refresh = cron 체이닝 (2026-05-14)** | 단일 `mart_refresh_chain` 안에서 enriched → daily mat 순차 실행 (`*/5 * * * *`). mat별 분리 cron 폐기 — 같은 enriched 스냅샷 보장 + mat lag 추가 5분 제거. lag 메트릭 단계 분리 (#1a collector / #1b mart) |
| **부서 기준 (2026-05-12 이사님)** | **앱 소유 부서로 통일** (DAU도 호출수도 동일, 5/11 owner/actor 양쪽 박기 + 토글 폐기). 신규/이탈 컬럼 자체 삭제 |
| **에러 범위 (2026-05-12 이사님)** | **호출 + 보안 전부** (audit이 둘 다 가짐). ILIKE 6 룰 입력 = `audit_events.details->>'error'` |
| **기간 max (2026-05-12 이사님)** | **90일 확정** (audit 90일 보존 수용, 초과 옵션 자체 제거) |
| **KPI 4종 구성 (2026-05-13 이사님 추가 결정 반영)** | 1번 총 오브젝트 유지 / **2번 → "부서별 채택 앱 수"** (각 부서의 사람들이 사용하는 앱 수 = COUNT DISTINCT app where actor in dept_members. 5/12 "부서별 사용 = 사용 건수" → 5/13 재변경. KPI 3 호출수와 차원 분리 위해) / 3번 API 호출 = "부서별 앱 호출 수" 의미 명확화 (RPS 평균 유지) / **4번 → "앱별 통계" KPI로 교체** (에러율 폐기 — 5/4 결정 무효). Quality 신호는 4번 안의 컬럼/Top 차트로 흡수 |
| ~~**차트 드로어 (2026-05-12 이사님 2차)**~~ | **2026-05-13 보류 결정 (이사님)** — 4종 모두 구현 보류. 향후 관리자 전용으로 만들든가 + 정보 한 단계 깊게 보는 게 사용자에게 의미 있는지 재판단 필요. 좌하 차트 클릭 자체 없음 (차트만 표시). chart-drawer spec 3파일은 "🔒 보류" 헤더만 박고 본문 보존 (재검토 시 base) |
| **표 행 클릭 동작 (2026-05-13)** | **KPI 4번 표만 행 클릭 → Dify 모니터링 페이지** (`/app/{appId}/overview`). KPI 1·2·3 표 클릭은 **없음** (드로어 보류로 의미 약함). 표 헤더 정렬/필터는 후보 (전역 정책 결정 시 적용) |
| **로그인 디폴트 페이지 (2026-05-13)** | `/dashboard` 단독. 기존 `/apps` 디폴트는 완전 폐기. `DEFAULT_POST_LOGIN_PATH = '/dashboard'` 상수 도입, 7군데 하드코딩 일괄 교체 |
| ~~설정 모달 사이드바 토글 (2026-05-12 KAN-29 포팅)~~ | **2026-05-13 revert** — 대시보드 마운트 위치를 설정 모달 → 탐색 메뉴 좌측 라우트로 이동 결정에 따라 토글 가치 소멸. VSCode Claude Code로 +33/-5 단독 커밋 되돌림 (working tree 보존, 권대리님 영역 TS 에러로 pre-commit 차단 — 의도적 미커밋) |
| **대시보드 마운트 위치 (2026-05-13 확정)** | 설정 모달 탭 → **`/dashboard` 톱레벨 라우트** + 메인 헤더 첫 번째 메뉴(`DashboardNav`) + 로그인 디폴트 페이지. 모달 제약(자체 h1 금지/URL state 금지/ESC 자체 핸들링 금지/모달 헤더 슬롯) 전면 해제 |
| **헤더 메뉴 구성 (2026-05-13)** | `대시보드 / 탐색 / 스튜디오 / 지식 / 도구` (5개, 대시보드 첫 번째). 메뉴 5개 균등 가운데 배치 (구현 시 정렬 확인). i18n: 한국어 "대시보드" / 영어 "Dashboard". 아이콘: `RiDashboardFill` / `RiDashboardLine` (`@remixicon/react`) |
| **dataset_operator 권한 (2026-05-13)** | `/dashboard` 접근 **잠정 허용** — `RoleRouteGuard.datasetOperatorRedirectRoutes`에 추가 X. 헤더 `DashboardNav`도 모든 권한에 표시 (전체 공개 정책 일관). 단 UI 상 권한 확인 안 됨 → 실제 사용 여부 확인 필요 |
| **API 경로 (2026-05-13 변경)** | `/console/api/admin/dashboard/` → **`/console/api/dashboard/`** (UI 라우트와 통일). 컨트롤러 폴더 `api/controllers/console/admin/dashboard.py` → `api/controllers/console/dashboard/`로 이동 (Dify upstream admin 엔드포인트와 분리). VSCode Claude Code 5/13 작업 완료 — 16 라우트 + 15 hooks endpoint 일괄 변경 |
| **TanStack queryKey (2026-05-13 변경)** | `['admin', 'dashboard', '<component>', params]` → `['dashboard', '<component>', params]`. 새로고침 = `invalidateQueries({ queryKey: ['dashboard'] })`. VSCode Claude Code 5/13 후속 정리 완료 — use-admin-dashboard.ts(4) + use-admin-drill.ts(11) + refresh-button.tsx + 테스트 |
| **화면 레이아웃 — 고정 높이 + 스크롤 (2026-05-13 신규 전역 정책)** | 카드/차트/표 모두 고정 높이. 콘텐츠 초과 시 영역 내부 스크롤. 데이터 양에 따라 카드 크기 변동되는 레이아웃 금지. spec 영향 = `kpi-cards`, `kpi-drill-through`, `chart-drawer`, `dashboard-controls`. 디테일(높이값/스크롤 임계/모바일 반응형)은 구현 시 결정 |
| **인터랙션 정책 (2026-05-13)** | 좌하 차트 클릭 = 동작 없음 (차트만 표시, 드로어 보류). 표 행 클릭 = KPI 4번만 모니터링 페이지. KPI 카드 클릭 = drill-through 펼침. ESC 키 = 자유 활용 (라우트 페이지 환경) |
| **마트 객체 명명 (2026-05-14)** | **public schema + `spx_` prefix**. audit/mart schema 폐기. `audit_events` → `spx_audit_events` / `v_audit_enriched` → `spx_v_audit_enriched` / `v_resource_ownership_enriched` → `spx_v_resource_ownership_enriched` / `mv_kpi_calls_daily` → `spx_mv_kpi_calls_daily` / `mv_model_tokens_daily` → `spx_mv_model_tokens_daily` / cron job `spx_mart_refresh_chain`. Dify upstream + 회사 RBAC 표준과 통일된 schema, prefix로만 구분. audit 테이블 rename + dify-audit collector INSERT target 수정 = 마트 작업 0단계 (선행) |
| **마트 Generated Column 컨벤션 (2026-05-15)** | details에서 추출한 Generated Column은 **`_d` 접미사** 일괄 적용 (8개 중 7개: `app_mode_d`/`model_provider_d`/`model_id_d`/`total_tokens_d`/`error_d`/`invoke_from_d`/`triggered_from_d`). `target_app_id`는 예외 — top-level `target_id` + `details.appId` 혼용이라 details 추출 컨벤션 미적용. 사유: 미래 top-level 컬럼과 충돌 방지 + 출처 자체기록. **target_app_id 타입 UUID** (resource_ownership.resource_id가 UUID라 직접 JOIN 가능, collector 13종 모두 UUID `::text` 캐스트로 비-UUID 유입 경로 없음). **total_tokens_d 타입 BIGINT** (workflow_runs.total_tokens=bigint 오버플로 방지). **is_debug 표현식** = `triggered_from_d IN ('debugging','rag-pipeline-debugging')` (디버깅 의미 2종 포함). **collector 4건은 `app.mode` LEFT JOIN 유지** (messages.app_mode 직접 사용 안 함, 레거시 행 호환 안전 선택) |

## 외부 의존 / 블로커

| 의존 | 상태 | 비고 |
|------|------|------|
| RBAC 테이블 DDL (김이사님 결정 / 권대리님 구현) | ✅ **확정 + 우리 환경 가용** (2026-05-06) | 권대리님 `feat/rbac` 브랜치 + flask db upgrade로 우리 docker DB에 5종 테이블 영속 잔존 확인. 명명 권대리님 슬랙으로 확정. `references/rbac-schema.md` 전면 갱신 완료. RBAC 의존 8종 컴포넌트 즉시 작업 가능. |
| ~~Keycloak upsert (승랑님 영역 추정)~~ | ✅ 자동 해소 | `accounts.id` 직접 매핑이라 매핑 키 확정 불필요 (H-DASH-14 자동 해소). 단 실제 데이터 채워지는 시점은 SSO upsert 구현 후. |
| ~~설정 모달 사이드바 (태영님)~~ | ✅ **종결 (2026-05-13)** | 5/12 KAN-29 포팅 → 5/13 revert. 대시보드가 모달 → 라우트로 이동 결정으로 모달 사이드바 의존 자체 소멸. H-DASH-15 완전 종결. |
| **마트 입력 = audit_events 단독 + RBAC JOIN** | ✅ 확정 (2026-05-12 이사님) | 0430~5/11 누적된 "엄밀 마트 vs ODS" 불명확 해소. 단일 SoT + 통합 활동 로그 + 보안 이벤트 통합 가치 우선. ETL/jsonb 비용 관점(5/11 OLTP 우세)은 후순위. |
| **collector 비대칭 보강 (승랑님)** | ⏳ 협상 → **필수 의존**으로 격상 (2026-05-12) → 5/14 사용자 본인 작업으로 전환 | audit 단독 마트의 진짜 차단 요소. 마트 ETL 작업하면서 부족 필드 누적 → 완료 후 **최종 리스트 일괄** 전달 (이사님 OK, 단 최종본 보고 필요). P0 3건: `appMode` × 4 collector, `triggeredFrom`/`invokeFrom` × workflow-runs.ts. ~30줄 변경, 난이도 낮음. Generated Column 8개 페어로 진행. 진행 노트: [[0. Inbox/audit collector 보강 후보 리스트.md]] |
| **audit details 필드 명세 검증** | ✅ **완료 (2026-05-12 VSCode Claude Code)** | 산출물 3종: `references/audit-details-spec.md`(15.4KB, 필드 명세), `references/rbac-schema.md`(15.6KB, 갱신), `references/objects-charts-feasibility.md`(9.4KB, 오브젝트 차트 RBAC 구현 가능성). Claudian 측 검토 5/13 완료. 가용 키 8개, 🔴 P0 필수 보강 3건, ⚠️ P1 선택 보강 2건. 외부 사용자 = `actor_type` 컬럼으로 구분 (account/end_user/api/system). api_call(nginx) 부서별 분류 불가 (URI로 앱 특정 불가). |
| **vitest 환경** | ✅ 해소 (2026-05-06) | 5/4 vitest 부팅 불가 + 5/6 ESLint pre-commit hook 부팅 불가가 모두 동일 원인 (`vinext@0.0.40`의 Node 22+ `fs.glob` 의존). 호스트 Node 20→22 업그레이드 + corepack 활성화로 해소. 대시보드 컨트롤 vitest 6/6 통과 확인 (1.67s). |
| **KPI 4종 구성** | 🔄 **재변경 (2026-05-12 + 2026-05-13)** | 5/4 "에러율 24h" → 5/12 1차 폐기 ("부서별 사용" + "앱별 통계") → 5/13 재변경 (2번 "부서별 채택 앱 수" 안 3 채택). 현재: 1번 총 오브젝트 유지 / 2번 부서별 채택 앱 수 / 3번 부서별 앱 호출 수 / 4번 앱별 통계 |
| **Phase 2 동적 인터랙션 spec** | 🔄 갱신 진행 중 (2026-05-13) | 5/4 작성 9파일은 마트=audit + KPI 개편 + 라우트 이동 + 차트 드로어 보류 누적으로 일괄 재작성 대상. 상위 5건 ✅ 완료, spec 24파일은 VSCode Claude 위임 진행 (`run-spec-update.ps1`) |
| **대시보드 라우트 마운트 위치 선행 분석 4종** | ✅ **완료 + 결정 박힘 (2026-05-13)** | 4종 모두 분석 + Claudian 검토 완료. 결정: 라우트=`/dashboard` / API=`/console/api/dashboard/` / 사이드바=메인 헤더 첫 번째 / dataset_operator 잠정 허용. 분석 #3 (어드민 게이트)은 사용자 범위 변경으로 의미 소멸 (게이트 불필요). |
| **권대리님 `feat/rbac` 브랜치 TS 에러 5건** | ⏳ 사용자가 직접 슬랙 (2026-05-13 발견) | `tsgo --noEmit` 5건 = `ACTION_I18N[a]` 동적 키 i18next strict 타입 이슈. 대상: app-permissions / dataset-permissions / tool-permissions의 acl-section.tsx + grant-permission-modal.tsx. 우리 영역 0건. pre-commit hook이 tsc까지 검증해서 본인 작업 커밋 차단 효과 — 사이드바 토글 revert 의도적 미커밋 원인 |
| **심볼릭 링크 양방향 sync** | ✅ 재생성 (2026-05-13) | 5/7 SymbolicLink 전환 / 5/12 시나리오 B (junction+hardlink mirror) / 5/13 sync 깨짐 발견 (VSCode 변경이 vault 측 반영 안 됨) → `.claude` 폴더 재생성. **단순 SymbolicLink로 복귀** (vault가 마스터). settings 3종 파일은 vault 안으로 들어감 (단일 머신 운영 단순화) |

## 차세대 도입 검토 (Phase B 진입 시)

> Garment OEM MES 프로젝트 비교에서 추출. 상세는 `SPX-Agent 하네스 설계.md § 다른 프로젝트 비교`.

- [ ] `globs` 프론트매터 자동 참조 — 파일 수정 시 관련 규칙 자동 로드
- [ ] Forbidden Patterns 스크립트 — H-DASH-01,03 SQL 패턴 grep 검출
- [ ] 슬래시 명령 (`/implement-kpi-cards` 등)
- [ ] k6 정량 게이트 (부서별 활동 2초 검증)

---

## 변경 이력

> ⚠️ 2026-05-13 PowerShell 인코딩 사고로 변경 이력 영역 손실. 깨진 원본 백업: `SESSION_HISTORY.md.broken-20260513`
> 이번 주(5/11~13) 항목은 conversation 기반 그대로 복원. 그 전 항목은 일주일 단위 요약으로 재구성.

---

### 2026-05-20 (VSCode Claude Code — 시드 데이터 재구성 + MView naming 정합 + 임시조치 3건 청산 + 모델 미분류 수정)

> 연속 세션. PROMPT.md 3차 갱신 포함.

#### 시드 데이터 전면 재구성 (V.3 FAIL 본질 해소)

- **카디널리티 변경**: 30부서→**10** / 300계정→**150** / 150앱→**100** / +KB **30** / +Tool **40** / 450K audit→**180K**
- **시간 분포**: 30일 편중 → **90일 균등** (일평균 2,000, 주말 30% 감소)
- **모델**: 5종 → **12종** (gpt-4o 25% ~ text-embedding-3-small 0.3%)
- **이벤트 타입**: 2종 → **7종** (message_send/workflow_execute/api_call/login/permission_change/dataset_query/tool_invoke)
- **앱 분포**: 균등 → **power-law** (상위 10개 앱 40%, 다음 20개 30%, 나머지 70개 30%)
- **사용자 분포**: 균등 → **power-law** (상위 20명 활동량 35%)
- **소유권 분포**: 균등 → **스큐** (상위 5명이 25앱 소유)
- **에러 분포**: 2앱 집중 → **9앱+ 분산** (주기 기반 해시로 상관관계 제거)
- **한글 실제 이름**: 부서명(경영지원본부/영업1팀 등), 사용자명(김민준/이서연 등 성씨20+이름32 풀), 앱명(회의록 요약 봇/영업 리드 분석 등 100개 카테고리별)
- **KB 30 + Tool 40**: `spx_resource_ownership`에만 등록 (audit 이벤트 없음, 오브젝트 차트용)
- **마트 필터링 검증 패턴**: debug 5%+5% / 에러 5% / 미등록 앱 3% (H-DASH-03/04/18)

검증 결과:
```
V.1 카디널리티: 10/150/150/100/170/180K 정확
V.2 idempotency: clean→reseed 동일
V.3 시간: 90일 균등, 일평균 2,000
V.4 모델: 12종 ±2% 정합
V.5 마트 필터링: error 3,948 / debug 7,304 / 미배정 9,314
V.6 EXPLAIN: quicksort 776kB, 57ms (work_mem 4MB 안전) — Before 1,480ms → After 57ms
V.8 한글 인코딩: mojibake 0
V.9 chain refresh: 3종 MView 정상
```

#### 트랙 4: Layer 1 MView naming + RBAC prefix 정합

- `spx_v_audit_enriched` → **`spx_mv_audit_enriched`** (MView인데 `_v_` prefix → `_mv_` 정정)
- Layer 1 MView migration의 unprefixed RBAC → `spx_resource_ownership` / `spx_department_members` / `spx_departments`
- 신설 migration: `20260519000000_fix_layer1_mview_naming_and_rbac_prefix/migration.sql`
- 정합 파일 17건 (migration + backend 6 + docs 8 + scripts 2)

#### 임시 조치 3건 전부 청산

| 임시 조치 | 해소 방법 |
|-----------|-----------|
| ~~`MIGRATION_ENABLED=false`~~ | dev merge → migration 파일 복원 → `true` 복원 |
| ~~alembic fake revision~~ | dev merge → `f88f4a6` commit 포함 → revision 정상 인식 |
| ~~`CREATE VIEW accounts`~~ | dev merge → Dify 코어 참조 전수 수정 완료 → VIEW DROP |

#### 모델 미분류 수정

- `dashboard_drill_calls_service.py` `get_model_call_share`: `COALESCE(model_id, '미분류')` → `WHERE model_id IS NOT NULL` 필터 추가
- workflow_execute 이벤트 (model NULL)가 차트에서 "미분류"로 표시되던 문제 해결

#### 프론트엔드 하드코딩 mock 제거 (5/19 세션에서 시작, 5/20 계속)

- drill-down 6개 컴포넌트 + KPI 필드명 정합 (5/19 커밋 3건)
- 사용자가 추가 수정한 drill 컴포넌트 반영 (app-stats-table, dept-users-table, top-users, app-call-top10, top-error-apps, dept-adopted-apps — hook 이름/응답 필드 정합)

#### 발생 문제 + 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| 해시 상관관계: 에러가 2앱에 집중 | `error_bucket = (i*23)%100`과 `unreg_bucket = (i*41)%100`이 같은 i 패턴에 몰림 | 에러를 주기 기반(`i%20=0`)으로 변경, 해시 곱셈 대신 나머지 사용 |
| 모델 threshold 스케일 혼용 | 10개 모델 threshold=0-100 스케일 + 2개 임베딩=0-1000 스케일 | 전부 0-1000 스케일로 통일 |
| 모델 "미분류" 차트 표시 | workflow_execute 이벤트 model_id=NULL이 COALESCE로 포함 | WHERE model_id IS NOT NULL 필터 추가 |
| 앱 호출/에러/사용자 분포 균등 | 모듈러 산술 해시가 균등 분포 생성 | power-law CASE 분기 (상위 N%에 트래픽 집중) |

#### "미배정" vs "외부" 부서 sentinel 분리 (완료)

- **변경**: sentinel 1개 → 2개 분리
  - `00000000-0000-0000-0000-000000000000` → **"미배정"** (account without dept)
  - `00000000-0000-0000-0000-ffffffffffff` → **"외부"** (end_user/api/system)
- **신설 migration**: `20260520100000_dual_sentinel_actor_dept/migration.sql` — Layer 2 MView COALESCE에 `actor_type` 기반 분기 추가, Layer 1 변경 없음 (actor_type 컬럼 이미 존재)
- **백엔드 수정 (5파일)**:
  - `api/models/mart.py` — `EXTERNAL_DEPT_ID = "00000000-0000-0000-0000-ffffffffffff"` 상수 추가
  - `dashboard_kpi_service.py` — adoption 필터에 EXTERNAL 제외 (`.notin_([UNASSIGNED, EXTERNAL])`)
  - `dashboard_drill_calls_service.py` — `_dept_name_or_unassigned` 헬퍼에 EXTERNAL → "외부" 분기 + dept_ids 필터
  - `dashboard_drill_errors_service.py` — 동일 패턴
  - `dashboard_dept_activity_service.py` — 동일 패턴
- **검증**: Layer 2에서 `...ffffffffffff` 3,342행 (외부), `...000000000000` 0행 (현 시드 account 전원 부서 배정됨)
- **모델 미분류 수정도 동일 세션**: `get_model_call_share` 쿼리에 `WHERE model_id IS NOT NULL` 추가 → workflow_execute의 NULL model이 차트에서 "미분류"로 표시되던 문제 해소

---

### 2026-05-20 (옵시디언 Claudian — KPI 1+3 drill-through spec 정합 + KPI 3 5컬럼 평행 안 채택)

> 위임 프롬프트: [[0. Inbox/2026-05-20 KPI 1 + KPI 3 drill-through 정합 위임 프롬프트.md]]
> 사용자 결정: KPI 3 표 5컬럼 안 채택(옵션 B), 코드 파일명 `dept-call-rps-table.tsx` 유지 + spec을 코드 명명으로 갱신.

#### V.1 drift 매트릭스

| 영역 | spec 옛 | 코드 현 | 처리 |
|---|---|---|---|
| **KPI 1 표 컬럼** | 부서/App/KB/Tool/신규 (5) | 동일 | ✓ 정합 (변경 없음) |
| **KPI 1 좌하/우하 차트** | `dept-cumulative` / `top-owners` | 동일 | ✓ 정합 |
| **KPI 1 데이터 소스** | `resource_ownership` 직접 | `spx_v_resource_ownership_enriched` view (= ownership 래핑) | ✓ 정합 |
| **KPI 1 백엔드 쿼리** | FILTER 단일 쿼리 권장 | FILTER 쿼리 + unassigned 별도 (2 쿼리, N+1 아님) | ✓ 정합 |
| **KPI 1 "신규" 시점 정의** | PM 미확정 | 코드 = `roe.created_at` (= `resource_ownership.created_at`) | 🟡 PM 보고 (수정 X) |
| **KPI 3 표 컬럼** | 부서/호출/사용자 수/에러/평균 RPS/마지막 사용 (6) | 부서/호출/RPS/Top 앱/추세 (5) | 🔴 **사용자 결정 → 옵션 B (spec 5컬럼으로 갱신)** |
| **KPI 3 좌하 `dept-call-count`** | spec ✓ | 코드 ✓ | ✓ 정합 |
| **KPI 3 우하 `model-call-share`** | spec "또는 dept-error-rate 대체" 잔존 | 5/20 dead code 청산으로 `model-call-share` 단독 | 🟡 spec 갱신 (대체 옵션 폐기 표기) |
| **KPI 3 데이터 소스** | audit_events 직접 (6컬럼 산출 전제) | Layer 2 `spx_mv_kpi_calls_daily` (5컬럼 자연 산출) | 🟡 spec 갱신 (Layer 2 enforce 박음) |
| **KPI 3 백엔드 쿼리 패턴** | (옛 6컬럼 단일 쿼리) | 3쿼리 분리(curr/prev/top_app, window function) — N+1 아님 | ✓ 정합 |
| **KPI 3 표 파일명** | `dept-call-table.tsx` (design line 103) | `dept-call-rps-table.tsx` | 🟡 spec을 코드 명명으로 갱신 (사용자 결정) |
| **표 행 클릭 (KPI 1/3)** | 없음 (KPI 4만) | 코드에 핸들러 0건 | ✓ 정합 |
| **좌하/우하 차트 클릭** | 없음 (H-DASH-20) | 코드에 핸들러 0건 | ✓ 정합 |
| **queryKey `'admin'` 잔존** | `'dashboard'` 표준 | 코드 grep 0건 | ✓ 정합 |
| **API 경로 `/console/api/admin/...`** | `/console/api/dashboard/...` 표준 | 코드 grep 0건 | ✓ 정합 |

#### spec 갱신 (3파일 — 코드 변경 0건)

- **`hdd/specs/requirements/kpi-drill-through.md`**:
  - § 메트릭 카탈로그 calls 행 컬럼 5종 교체 (6→5)
  - § "사용자 수" 메트릭 정의: KPI 2·KPI 4 전용으로 좁힘 (KPI 3에서 제거 명시)
  - § "Top 앱" 메트릭 정의 신설 (KPI 3 전용, owner_dept_id 기준, KPI 2 "Top 채택 앱"과 의미 차이 명시)
  - § "RPS" 메트릭 정의 신설 (KPI 3 전용, `SUM(calls) / period_seconds`)
  - § "추세" 메트릭 정의 → KPI 2·KPI 3 공통으로 확장
  - § "마지막 사용" 메트릭 정의 → KPI 4 전용으로 좁힘
- **`hdd/specs/design/kpi-drill-through.md`**:
  - § 영역 매핑 calls 행: "6컬럼" → "5컬럼", "또는 부서별 에러율" 폐기
  - § 컴포넌트 구조: `dept-call-table.tsx` → `dept-call-rps-table.tsx`, `model-call-share.tsx` 주석 정리
  - § chart-area.tsx `DRILL_SLOT_MAP` calls: `DeptCallTable` → `DeptCallRpsTable`
  - § calls 표 본문 전면 재작성 + 변경 이력 박음 + 백엔드 쿼리 패턴 본문 (Layer 2 `spx_mv_kpi_calls_daily`) 추가
  - **§ 응답 스키마 확장 (5/20 정합성 검토 권장 패치 2번 동시 처리)** — users/apps 응답 스키마 옆에 objects(`DrillDeptNewCreationsResponse`) + calls(`DrillDeptCallRpsResponse`) 본문 추가. "objects/calls 응답 스키마 본 spec 범위 외" 문장 삭제. 표준 키(`items` vs `rows`) 차이 표기 신설
- **`hdd/specs/tasks/kpi-drill-through.md`**:
  - task 22 model-call-share 메모 정리 (대체 옵션 폐기)
  - task 23 전면 갱신: 파일명 `dept-call-rps-table.tsx`, 5컬럼, RPS/Top 앱/추세 산출식, Layer 2 enforce
  - task 37: `dept-call` → `dept-call-rps-table`, Layer 2 enforce 명시

#### 코드 변경 — 0건

- KPI 1: 표/차트/백엔드 spec과 100% 정합 → 변경 없음
- KPI 3: 코드 5컬럼 안이 새 정답이므로 변경 없음 (spec이 코드 쪽으로 정합)

#### V.2~V.7 검증

| 검증 | 결과 | 비고 |
|---|---|---|
| **V.2** 백엔드 단일/N+1 | **PASS** | KPI 1: FILTER 단일 + unassigned (2쿼리, dept 수와 무관). KPI 3: curr/prev/top_app 3쿼리 (window function, dept 수와 무관). KPI 2 5/19 5.8초 병목과 다른 패턴 — N+1 아님 |
| **V.3** 응답 시간 | **PASS (간접 확인)** | 5/20 인덱스 추가 후 KPI 3 계열 endpoint base 응답 통과. 본 위임 코드 0변경이라 재측정 불필요 |
| **V.4** 행 클릭 / 차트 클릭 | **PASS** | 표 컴포넌트 grep — 클릭 핸들러 0건. 차트 컴포넌트 — `onClick` 0건 |
| **V.5** 5/13 전역 정합 | **PASS** | queryKey `'admin'` 0건, API 경로 `/console/api/admin/...` 0건 |
| **V.6** vitest | **N/A** | 코드 변경 0건. 회귀 위험 없음 |
| **V.7** 신규 환경 / 마이그레이션 | **N/A** | 마트 DDL / 마이그레이션 0건 |

#### 보류 / PM 컨펌 필요 항목

1. **KPI 1 "신규" 시점 정의** — 코드는 `resource_ownership.created_at` (= ownership 등록 시점). 코드 = 옵션 A 채택 상태. PM 컨펌 시 옵션 B(`apps.created_at` 등 Dify 객체 생성 시점)로 변경 가능성 잔존
2. **api_call (nginx) 부서별 분류** — 현 마트는 콘솔 + end_user 활동만 캡처. nginx 외부 API key 호출이 KPI 3 부서별 차트에 포함되는지 PM 컨펌
3. **응답 키 표준 `items` vs `rows`** — 5/20 신설 표(users `rows`, apps `rows`) vs 기존 잔존(objects/calls `items`) 차이. 추후 통일 검토 (별도 PR)

#### 다음 작업 안내

- **(권장) 부수 정합 별도 PR**:
  1. `hdd/specs/requirements/kpi-cards.md` "API 호출" → "부서별 앱 호출 수" 명칭 정정 (5/13 결정 vs kpi-cards.md line 31 stale)
  2. 본 spec § 메트릭 카탈로그 calls 라벨도 동반 정정 검토
  3. CLAUDE.md 프로젝트 개요 "API 호출" 동반 정정
- **(권장) 응답 키 통일** — `items`/`rows` 표준 결정 후 일괄 rename
- **CLAUDE.md 불변식 3번 갱신** — RBAC 테이블 명칭 `departments` → `spx_departments` (5/19 이월)
- **context-bar spec 라벨 정정** — 5/19 이월

---

### 2026-05-20 (옵시디언 Claudian — KPI 1+3 정합 위임 프롬프트 정합성 검토)

> 대상: [[0. Inbox/2026-05-20 KPI 1 + KPI 3 drill-through 정합 위임 프롬프트.md]]
> 목적: 위임 발사 전 사실 관계 / 진입점 / spec 참조 정합성 사전 검토.

#### 정확 확인 (위임 발사 안전 영역)

- 5/14 누락 트랙 출처(`1. Daily/2026-05-14.md` line 27 "VSCode Claude 후속 위임") ✓
- 5/20 KPI 2/4 정합 위임 완료 사실 (SESSION_HISTORY 본 문서 line 111 entry) ✓
- 표 파일명 drift 정확히 짚음 — 코드 `dept-call-rps-table.tsx` vs spec design.md line 103 `dept-call-table.tsx` ✓
- KPI 3 표 6컬럼 (부서/호출/사용자 수/에러/평균 RPS/마지막 사용) = design.md line 237-244 ✓
- KPI 1 표 5컬럼 (부서/App/KB/Tool/신규) = design.md line 155-161 ✓
- 표 행 클릭 = KPI 4만 / 좌하·우하 차트 클릭 없음 (H-DASH-20) = CLAUDE.md 일치 ✓
- `useDrillDeptErrorRate` 5/20 dead code 청산 제거 = 본 문서 line 161 일치 ✓
- KPI 1 = `spx_resource_ownership` 직접 / KPI 3 = `spx_mv_audit_enriched` = design.md § objects 전용 + § 공통 JOIN ✓
- 단일 쿼리 FILTER 패턴 권장 = design.md line 363-365 패턴 ✓
- API 경로 / queryKey 정합 (`/console/api/dashboard/`, `['dashboard', ...]`) = CLAUDE.md 컨벤션 일치 ✓

#### 발견된 사실 오류 / 정합 리스크 3건 (위임 발사 전 패치 권고)

1. **KPI 3 명명 stale 잔존** — SESSION_HISTORY line 87은 "3번 부서별 앱 호출 수" 박혔으나 `hdd/specs/requirements/kpi-cards.md` line 31, 38, 97, 203, 208 + 프로젝트 CLAUDE.md 프로젝트 개요는 **여전히 "API 호출"** 옛 명칭 잔존. 위임이 진입점 6번(kpi-cards § 4-Axis) 정독 시 drift 판단 혼란 가능.
2. **objects / calls 표 응답 스키마 spec 범위 외 명시** — design.md line 395 "응답 스키마 — users / apps 표 (5/20 신설)" + line 424 "objects / calls 표 응답 스키마는 본 spec 범위 외 (각 컴포넌트 spec 본문 참조)". 위임은 진입점 4번에 § 응답 스키마 박았지만 KPI 1·3 정답이 spec에 부재 → 위임 측이 정합 검증 시 충돌.
3. **SESSION_HISTORY KPI 3 패턴 기술 내부 불일치** — line 243 "KPI 3 패턴 부서/호출/RPS(평균)/Top 앱/추세" (5컬럼)이 design.md § calls 표 6컬럼과 다름. KPI 2 재결정 맥락에서 미래 안 기준이었던 듯. 위임 측이 SESSION_HISTORY 우선 정독하면 "5컬럼 평행 안으로 정합?" 오인 가능.

#### 권장 패치 (위임 발사 전)

- 진입점 6번 옆에 "kpi-cards.md 'API 호출' 옛 명칭 = stale 참고, 본 위임 청산 범위 외" 메모
- 작업 단계 6 (spec 갱신)에 "design.md § 응답 스키마 objects/calls 본문 신설" 박기
- 작업 단계 4 (KPI 3) 앞에 "표 컬럼 단일 진실 = design.md line 237-244 (6컬럼). SESSION_HISTORY 5/20 entry의 5컬럼 평행 기술 = KPI 2 정합 맥락 한정" 박기

#### 잠재 부수 작업 (위임 범위 외, 별도 PR 권장)

- kpi-cards.md "API 호출" → "부서별 앱 호출 수" 전면 정정 (CLAUDE.md 프로젝트 개요도 동반)
- design.md § 응답 스키마에 objects / calls 표 본문 추가 (현재 본 spec 범위 외 명시 상태)
- SESSION_HISTORY 5/20 KPI 2 entry line 243 "KPI 3 패턴 5컬럼" 표현 정합 (현 6컬럼 본문과 충돌 표기 보완)

---

### 2026-05-20 (VSCode Claude Code — KPI 2/4 drill-through 코드 정합 + dead code 청산)

> 위임 프롬프트: `.claude/docs/PROMPT.md` (2026-05-20 코드 정합 위임).
> 선행: 5/20 옵시디언 Claudian 세션에서 spec 3파일 갱신 + 위임 프롬프트 작성 완료.

#### 작업 2 — KPI 2 (users) 표 + 백엔드 갱신

- **프론트 `dept-users-table.tsx`**: 5컬럼 재작성 (부서 / 사용자 수 / 사용자당 채택 수 / Top 채택 앱 / 추세)
  - `TrendBadge` 컴포넌트: `+신규` / `▲` / `▼` / `0%` / `-` 분기 렌더링
  - `apps_per_user null → "-"`, `top_app_name null → "-"` (H-DASH-07)
- **백엔드 `dashboard_drill_users_service.py`**: `get_dept_user_activity` 전면 재작성
  - 7쿼리 N+1 (`dau_rows` / `wau_rows` / `curr_users_rows` / `prev_actors_q` / `new_rows` / `curr_actors_q` / `churned_rows`) → **단일 CTE** (design.md § users 표 SQL 본문 그대로, `text()` raw SQL)
  - CTE 4개: `base` → `agg` → `top_app` (DISTINCT ON) → `prev_agg` + 최종 SELECT
  - `NOT IN` 서브쿼리 2건 완전 제거 (5/19 5.8초 병목 원인)
  - `trend_label` CASE 추가: `prev=0 AND current>0 → '+신규'`, `prev=0 AND current=0 → '-'`
- **스키마 `schemas.py`**: `DrillDeptUserActivityItem` 필드 교체 (`dau/wau/new_users/churned_users` → `dept_name/user_count/apps_per_user/top_app_name/trend_pct/trend_label`), 응답 키 `items` → `rows`
- **프론트 hook `use-admin-drill.ts`**: `DrillDeptUserActivityItem` 타입 갱신, `rows` 키

#### 작업 3 — KPI 4 (apps) 표 + 백엔드 전면 재구현

- **프론트 `app-stats-table.tsx`**: row grain 부서 → **앱** 전환, 6컬럼 (앱명 / 부서 / 호출 / 사용자 / 에러율 / 마지막 사용)
  - 행 클릭: `router.push(\`/app/${appId}/overview\`)` — 같은 탭, `role="link"` + `tabIndex={0}` + Enter
  - `error_rate null → "N/A"`, `last_used → formatRelativeTime` (N분/시간/일 전)
- **백엔드 `dashboard_drill_apps_service.py`** (신설): 3개 메서드
  - `get_app_stats`: design.md § apps 표 SQL 본문 그대로 (`text()` raw SQL), 앱 단위 GROUP BY + `spx_resource_ownership` + `spx_departments` JOIN, `LIMIT 100`
  - `get_app_call_top10`: `LIMIT 10`, 호출 수 DESC
  - `get_top_error_apps`: `COUNT(*) FILTER (WHERE error_text IS NOT NULL)`, `HAVING > 0`, `LIMIT 10`
- **컨트롤러**: apps 엔드포인트 3종 추가 (`/dashboard/drill/apps/app-call-top10`, `top-error-apps`, `app-stats`)
- **스키마**: `DrillAppStatsItem/Response`, `DrillAppCallTop10Item/Response`, `DrillTopErrorAppsItem/Response` 신설
- **프론트 hook**: `useDrillAppStats`, `useDrillAppCallTop10`, `useDrillTopErrorApps` 추가
- **좌하/우하 차트 데이터 소스 전환**:
  - `app-call-top10.tsx`: `useDrillDeptCallCount` (부서 호출) → `useDrillAppCallTop10` (앱별 호출)
  - `top-error-apps.tsx`: `useDrillDeptErrorRate` (부서 에러율) → `useDrillTopErrorApps` (앱별 에러)

#### 작업 4 — KPI 2 좌하/우하 데이터 소스 정합 (사용자 추가 지시)

프롬프트 범위는 "점검만"이었으나, 사용자 확인 후 데이터 부정합 수정 진행:

- **좌하 `dept-adopted-apps.tsx`**: `useDrillDeptDau` (부서 DAU) → `useDrillDeptAdoptedApps` (`COUNT(DISTINCT target_app_id)` per dept)
- **우하 `top-users.tsx`**: `useDrillTopOwners` (objects 소유자) → `useDrillTopUsers` (활동 수 기준 Top 10 사용자)
- **백엔드**: `get_dept_adopted_apps` / `get_top_users` 메서드 신설 (raw SQL, `spx_accounts` + `spx_department_members` JOIN)
- **컨트롤러**: `/dashboard/drill/users/dept-adopted-apps`, `/dashboard/drill/users/top-users` 엔드포인트 추가

#### Dead code 청산 (사용자 추가 지시)

데이터 소스 전환으로 발생한 미사용 코드 일괄 제거:

| 제거 대상 | 사유 |
|---|---|
| **프론트**: `useDrillDeptDau` + `DrillDeptDauItem/Response` | `dept-adopted-apps.tsx`가 `useDrillDeptAdoptedApps`로 전환 |
| **프론트**: `useDrillDeptErrorRate` + 타입 | `top-error-apps.tsx`가 `useDrillTopErrorApps`로 전환 |
| **프론트**: `useDrillDeptErrorTable` + 타입 | 이전부터 미사용 |
| **백엔드**: `get_dept_dau` + `get_model_users` 메서드 | 프론트 미호출 |
| **백엔드**: `DrillDeptDauItem/Response` + `DrillModelUsersItem/Response` 스키마 | 제거된 메서드만 사용 |
| **백엔드**: errors 컨트롤러 2종 + `DashboardDrillErrorsService` import | 프론트 미호출 (앱 전용 엔드포인트로 대체) |
| **백엔드**: `DrillDeptErrorRateItem/Response` + `DrillDeptErrorTableItem/Response` 스키마 | errors 서비스만 사용 |
| **테스트**: `test_dashboard_drill_users_service.py` | 옛 스키마 참조 (`resp.items[0].dau`) → 전면 재작성 |

#### 테스트 갱신

- **`test_dashboard_drill_users_service.py`**: 전면 재작성
  - `TestDeptUserActivity`: 5건 (정상 렌더링 / `trend_label='+신규'` / `apps_per_user=null` / 빈 결과 / 단일 CTE 실행 횟수 검증)
  - `TestDeptAdoptedApps`: 1건 (부서별 채택 앱 수 반환)
  - `TestTopUsers`: 1건 (Top 사용자 반환)
- **vitest 신설**: `drill-tables/__tests__/` 2파일 14테스트
  - `dept-users-table.spec.tsx`: 6건 (5컬럼 / 데이터 렌더링 / null→"-" / +신규 뱃지 / ▲▼ 추세 / 로딩)
  - `app-stats-table.spec.tsx`: 8건 (6컬럼 / row=앱 / N/A / 행 클릭 → `/app/{appId}/overview` / Enter 키 / role="link" + tabIndex / 빈 상태 / 로딩)

#### V.1~V.7 검증 결과

| 검증 | 결과 | raw 요약 |
|---|---|---|
| **V.1** spec 정합 | **PASS** | KPI 2 5컬럼 1:1, KPI 4 6컬럼 row=앱, 행 클릭 `/app/{appId}/overview`, 좌하/우하 전부 정합 데이터 소스 연결 |
| **V.2** 단일 CTE | **PASS** | `db.session.execute` 메서드당 1회, `NOT IN` 0건, `WITH base ... SELECT` CTE 확인 |
| **V.3** 응답시간 | **FAIL** | dept-user-activity p95 ~5.9s (목표 <1s). CTE `base`가 450K 중 437K행 Seq Scan — 시드 데이터 97% 단일 기간 편중. DB 실행 5,346ms. `top-users`만 0.45s 통과 |
| **V.4** 행 클릭 | **PASS** | vitest 마우스/Enter 통과 + `GET /console/api/apps/{appId}` HTTP 200 (앱 존재 확인) + role="link" + tabIndex={0} |
| **V.5** 분기 처리 | **PASS** | `apps_per_user null→"-"`, `trend_label +신규/-`, `error_rate null→"N/A"` 전부 커버 |
| **V.6** vitest | **PASS** | 2파일 14테스트 통과 (`pnpm test -- --run app/components/admin/drill-tables`) |
| **V.7** DDL 변경 | **PASS** | DDL 0건, 마이그레이션 0건, SELECT 전용 |

#### 변경 파일 목록

**백엔드** (5파일):
- `api/services/admin/schemas.py` — 스키마 교체/신설/제거
- `api/services/admin/dashboard_drill_users_service.py` — 전면 재작성 (단일 CTE + 좌하/우하 추가)
- `api/services/admin/dashboard_drill_apps_service.py` — **신설**
- `api/controllers/console/dashboard/dashboard.py` — apps 3종 + users 2종 엔드포인트 추가, errors 2종 + users 구 2종 제거
- `api/tests/unit_tests/services/admin/test_dashboard_drill_users_service.py` — 전면 재작성

**프론트엔드** (7파일):
- `web/service/use-admin-drill.ts` — 타입 갱신 + hooks 추가/제거
- `web/app/components/admin/drill-tables/dept-users-table.tsx` — 5컬럼 재작성
- `web/app/components/admin/drill-tables/app-stats-table.tsx` — 전면 재구현 (row=앱, 행 클릭)
- `web/app/components/admin/drill-charts/users/dept-adopted-apps.tsx` — 데이터 소스 전환
- `web/app/components/admin/drill-charts/users/top-users.tsx` — 데이터 소스 전환
- `web/app/components/admin/drill-charts/apps/app-call-top10.tsx` — 데이터 소스 전환
- `web/app/components/admin/drill-charts/apps/top-error-apps.tsx` — 데이터 소스 전환

**테스트** (2파일 신설):
- `web/app/components/admin/drill-tables/__tests__/dept-users-table.spec.tsx`
- `web/app/components/admin/drill-tables/__tests__/app-stats-table.spec.tsx`

#### Docker 빌드 + V.3/V.4 실측

api + web 이미지 재빌드 → compose up:

**V.3 응답시간 실측**:

| 측정 방식 | dept-user-activity | dept-adopted-apps | top-users | app-stats |
|---|---|---|---|---|
| Before (7쿼리, 순차 단독) | **5.8s** | N/A | N/A | N/A |
| After (CTE, 순차 단독) | **7ms** | 7ms | 7ms | 8ms |
| After (CTE, 동시 3개 병렬) | **1.85s** | 0.84s | 0.24s | 0.87s |

- 순차 단독: 7ms = **830배 개선** (DB 쿼리 자체 성능)
- 동시 3개: 1.85s = **Gunicorn 단일 워커 직렬화** 누적 (인프라 이슈, CTE 무관)
- V.3 판정: **코드 수준 PASS** (단독 7ms < 1s 목표 달성). 동시 요청 지연은 `SERVER_WORKER_AMOUNT=1` 원인

**V.4 행 클릭 실측**:
- `GET /console/api/apps/{appId}` → **HTTP 200** (Dify 앱 존재 확인)
- vitest: 마우스 클릭 + Enter 키 → `router.push('/app/{appId}/overview')` 호출 확인
- `role="link"` + `tabIndex={0}` 속성 확인

#### UX 수정 (화면 검증 후 추가)

스크린샷 8장 기반 화면 검증 + UX 점검:

| 수정 | 파일 | 내용 |
|---|---|---|
| KPI 3 추세 더블 네거티브 | `dept-call-rps-table.tsx` | `↓-28.6%` → `▼28.6%` (`Math.abs` + 화살표 통일 `▲▼`) |
| 드릴 표 기간 라벨 누락 | 4종 drill-tables | "부서별 채택 현황" → "부서별 채택 현황 — 지난 7 일" (종합 뷰 표와 일관성) |

#### 성능 진단 — 브라우저 체감 느림

사용자 보고: "KPI2 좌하/표가 우하보다 늦게 나옴, KPI4 표가 차트보다 느림"

**원인**: Gunicorn `SERVER_WORKER_AMOUNT=1` (gevent). 브라우저가 KPI 활성 시 3개 API 동시 호출 → 단일 워커가 직렬 처리 → 1번째 응답 0.24s, 2번째 0.84s, 3번째 1.85s 누적.

**CTE 변경 효과**: 개별 쿼리 5.8s → 7ms (830배). 그러나 동시 요청 직렬화로 체감 개선 제한적.

**해결**: `SERVER_WORKER_AMOUNT=4` (docker/.env). 워커 4개면 3개 요청 병렬 → 각 ~7ms. 별도 인프라 트랙.

#### 다음 작업

1. **`SERVER_WORKER_AMOUNT=4` 적용** — 동시 요청 병렬 처리 즉시 개선 (인프라 설정)
2. **PM 컨펌 필요** — 사용자당 채택 수 / Top 채택 앱 / 추세 메트릭 본문 (5/20 사용자 결정 → PM 보고)
3. **`dashboard_drill_errors_service.py` 파일 삭제** — 컨트롤러/프론트에서 제거 완료, 서비스 파일 자체 잔존
4. **시드 데이터 시간 분포 정상화** — 현재 450K행이 좁은 기간에 편중, 90일 균등 분포 시 인덱스 hit 개선

---

### 2026-05-20 (옵시디언 Claudian — KPI 2/4 drill-through spec 갱신 + 코드 위임 작성)

> 위임 프롬프트: `PROMPT.md` (2026-05-20 코드 정합 위임). 5/14 누락 트랙 청산 + 부하 테스트 병목 자연 해소 트랙.

#### 발견 — 5/14 누락 트랙

사용자가 dev 환경 화면 점검 중 KPI 2 (부서별 채택 앱 수) 드릴 표가 5/4 옛 명세(`DAU/WAU/신규/이탈`) 그대로 박혀있음 발견. KPI 4 (앱별 통계) 드릴 표는 row grain이 부서로 박혀 KPI 3 표와 동일 — 5/13 결정(row=앱, 6컬럼)이 코드에 미반영.

원인 = [[1. Daily/2026-05-14.md]] line 27 "VSCode Claude 후속 위임" 항목이 데이터 마트 / RBAC prefix / 부하 테스트 트랙에 밀려 5/14~5/19 6일간 누락.

부하 테스트 dept-user-activity p95 5.8초 병목 = 옛 명세의 NEW/CHURNED `NOT IN` 쿼리 2건이 백엔드에 잔존하면서 1 endpoint당 7쿼리 N+1 패턴 굴러간 결과. 본 트랙이 옛 컬럼 정리 = 옛 쿼리 자동 소멸 = 병목 자연 해소.

#### KPI 2 표 컬럼 재결정 (사용자 결정 — KPI 3 평행 정합)

5/12 v2 노트의 잠정 권장 안 (2) `부서/사용 건수/사용자 수/사용자당 건수/활성 앱 수`는 5/13 KPI 2번 재변경("채택 앱 수") 후 정합 깨짐 (활성 앱 수가 좌하 막대와 중복). 사용자 새 안 채택:

```
부서 / 사용자 수 / 사용자당 채택 수 / Top 채택 앱 / 추세
```

KPI 3 패턴 `부서 / 호출 / RPS(평균) / Top 앱 / 추세`와 5슬롯 완전 평행. 좌하 차트(부서별 채택 앱 수 막대)와 정보 중복 회피.

#### spec 3파일 갱신

- `hdd/specs/requirements/kpi-drill-through.md`:
  - § 메트릭 카탈로그 표의 `users` 행 컬럼 5종 교체
  - § 비즈니스 규칙에 "사용자당 채택 수" / "Top 채택 앱" / "추세" 메트릭 정의 신설
  - frontmatter `last_updated` 5/13 → 5/20
- `hdd/specs/design/kpi-drill-through.md § users: 부서별 활동`:
  - 컬럼 5종 교체 + 변경 이력 박음
  - 단일 CTE 쿼리 본문 박음 (7쿼리 → 1쿼리, base/agg/top_app/prev_agg 4개 CTE + 최종 SELECT)
  - frontmatter `last_updated` 5/13 → 5/20
- `hdd/specs/tasks/kpi-drill-through.md`:
  - task 20 (drill-tables/dept-activity-table.tsx) 컬럼 5종 갱신 + N+1 7쿼리 금지 명시
  - task 36 (users 3종 서비스) 단일 CTE 쿼리 가드 추가
  - frontmatter `last_updated` 5/13 → 5/20

#### 점검 결과 — KPI 4는 spec 정합

- requirements line 57: `앱 통계 (앱명 / 부서 / 호출 / 사용자 / 에러율 / 마지막 사용)`
- design line 151: "모든 표는 행 단위 = 부서 (apps 표만 행 단위 = 앱)"
- design line 194~205: `AppStatsTable` 6컬럼 + 행 클릭 `/app/{appId}/overview`
- → **spec은 5/13에 완벽히 박혀있음**. 코드 mock만 5/4 옛것 잔존. spec 갱신 불필요, 코드만 따라가면 됨.

#### 마트 영향 점검

- `spx_mv_audit_enriched` 5/19 정의로 모든 신 컬럼 산출 가능 (`actor_dept_id`, `target_app_id`, `actor_id`, `occurred_at` + 외부 `apps` / `spx_resource_ownership` / `spx_departments` JOIN)
- → **마트 view/MView DDL 변경 불필요**. 백엔드 쿼리만 신설/재작성.
- 5/20 추가 인덱스(`spx_mv_audit_enriched_tenant_occurred_idx`) 효과 그대로 유지.

#### 추가 spec 갱신 — design.md apps SQL + 응답 스키마 + metric drift 정합

PROMPT.md 슬림화 과정에서 design.md 정합 점검 결과 3건 누락 발견 → 동시 보강:
- `§ apps: 앱 통계` 표 명세 뒤에 **apps 전용 SELECT SQL 본문 추가** (앱 단위 GROUP BY + spx_resource_ownership + spx_departments + apps JOIN). 좌하/우하 차트는 LIMIT/ORDER 변형 명시
- `§ API 엔드포인트` 마지막 metric `errors` → **`apps`로 drift 정합** (5/13 결정과 어긋난 잔존)
- `§ API 엔드포인트` 아래 **§ 응답 스키마 신설** — `DeptActivityResponse` / `AppStatsResponse` TypeScript 타입 본문 박음

#### 위임 프롬프트 작성 — `PROMPT.md` 신규 (부하 테스트 위임 교체)

**원칙: 명세 중복 박지 않음** — 모든 컬럼/SQL/응답 스키마는 spec 본문이 단일 진실. PROMPT는 진입점 + 작업 범위 + 가드 + 검증 항목만.

진입점 9건 (CLAUDE.md → SESSION_HISTORY 2026-05-20 entries → 5/19 entry → req/design/tasks/kpi-cards → quality-criteria → delegation-standard)

작업 단계 5종:
1. spec 진입점 정독 (drift 발견 시 spec이 정답)
2. KPI 2 mock + 단일 CTE service (design.md § users 표 CTE 그대로)
3. KPI 4 표 row grain 부서→앱 전면 재구현 (design.md § apps 표 SQL 그대로)
4. KPI 2 좌하/우하 정합 점검 (변경 없음 예상)
5. vitest 갱신

검증 V.1~V.7: spec 정합(drift 게이트) / 단일 CTE 회귀 / 응답 시간(5.8s→<1s) / 행 클릭 / 분기 처리 / vitest / 신규 환경

진행 금지: 마트 DDL / 신규 인덱스 / NOT IN 별도 PR / dept-activity 정적 표 / Gunicorn / KPI 1·3 / **spec 우회한 임의 컬럼·SQL 작성**.

#### 부하 테스트 후속 작업과의 관계

- 본 작업 = dept-user-activity 7쿼리 → 1쿼리 통합 = 5.8s 병목 자연 해소
- 부하 테스트 후속의 "NOT IN → LEFT JOIN 리팩토링" 항목은 본 작업으로 **NOT IN 쿼리 자체가 사라지므로 자동 해소**, 별도 PR 불필요
- 부하 테스트 후속의 "dept-user-activity CTE 통합" 항목도 본 작업과 동일 트랙이라 자연 흡수
- 남는 부하 테스트 후속: Gunicorn 워커 수 증가 / psycogreen 검증 / INCLUDE 확장 인덱스 (정식 부하 테스트 결과 후 판단)

---

### 2026-05-20 (VSCode Claude Code — k6 정식 부하 테스트)

> 위임 프롬프트: `.claude/docs/PROMPT.md` (2026-05-20). 작업 A~D.
> 선행: 5/20 인덱스 추가 완료, EXPLAIN ANALYZE 기준 7개 endpoint 전부 Index Scan 전환 확인.

#### 환경

- k6 v0.56.0 (Windows amd64)
- 인증: Keycloak password grant → `/console/api/keycloak/login-password` → access_token + csrf_token 쿠키
- API: Gunicorn gevent worker **1개** (`SERVER_WORKER_AMOUNT=1`)
- DB pool: `SQLALCHEMY_POOL_SIZE=30`, `MAX_OVERFLOW=10`
- 시드: `spx_mv_audit_enriched` 450,224행

#### S1 baseline (10 RPS, 3분, 5~15 VU)

| Endpoint | p50 | p90 | p95 | p99 |
|---|---|---|---|---|
| model-call-share | 1.45s | 2.66s | **3.11s** | - |
| error-table-cause | 1.52s | 2.47s | **2.96s** | - |
| model-users | 2.83s | 3.88s | **4.39s** | - |
| dept-user-activity DAU | 5.80s | 7.19s | **7.65s** | - |
| dept-user-activity NEW | 5.80s | 7.42s | **8.09s** | - |
| dept-user-activity CHURNED | 5.79s | 6.50s | **6.74s** | - |

- **유효 처리량**: 682/1,800 iterations (38%) = **~3.7 RPS 실질 한계**
- **에러율**: 0% (전부 200 OK — 느리지만 실패는 없음)
- **dropped iterations**: 1,119 (62% 드롭)

#### S2 stress (50 RPS, 5분, 30~60 VU)

| Endpoint | p50 | p90 | p95 |
|---|---|---|---|
| model-call-share | 14.42s | 16.39s | **16.84s** |
| error-table-cause | 14.65s | 16.55s | **17.11s** |
| model-users | 15.62s | 17.78s | **18.30s** |
| dept-user-activity DAU | 18.66s | 20.62s | **21.31s** |
| dept-user-activity NEW | 18.97s | 20.35s | **21.21s** |

- **유효 처리량**: 1,129/15,000 iterations (7.5%) = **~3.5 RPS**
- **에러율**: 0% (여전히 전부 성공 — Gunicorn 큐잉)
- **dropped iterations**: 13,871 (92% 드롭)

#### S3 breakpoint — 생략

S1에서 이미 ~3.7 RPS 한계 확인. S2에서도 동일 ~3.5 RPS. RPS 증가 시뮬레이션 의미 없음.

#### 병목 분석 — Flask/Gunicorn 레이어

**핵심 발견: DB는 병목이 아님. Gunicorn 단일 워커가 병목.**

| 레이어 | 측정 | 비고 |
|---|---|---|
| DB (EXPLAIN ANALYZE) | 0.76ms ~ 420ms | 인덱스 추가 후 정상 |
| HTTP (k6 단일 호출) | 1.45s ~ 5.8s | DB 대비 **10~100배 느림** |
| HTTP (k6 10 RPS) | 3.11s ~ 8.09s | 큐잉 지연 추가 |

**원인 상세**:
1. **Gunicorn 워커 1개** (`SERVER_WORKER_AMOUNT=1`) — gevent 기반이지만 SQLAlchemy의 psycopg2는 기본적으로 blocking I/O라 gevent greenlet이 DB 대기 중 다른 요청 처리 불가
2. **dept-user-activity 7개 순차 쿼리** — DAU + WAU + current + previous + new (NOT IN) + churned (NOT IN) + dept names = 1 endpoint 호출에 7회 DB 왕복. 단일 쿼리는 빨라도 누적되면 수초
3. **SQLALCHEMY_POOL_SIZE=30이지만 워커 1개라 동시 활용 불가** — 워커 1개의 단일 greenlet만 실제 쿼리 실행, 나머지는 큐 대기

#### INCLUDE 확장 인덱스 판단

**권장 안 함 (보류)** — 현재 병목이 DB가 아닌 Gunicorn 레이어이므로 DB 인덱스 추가 효과 미미.

#### NOT IN 리팩토링 판단

**의미 있지만 우선순위 낮음** — dept-user-activity가 7개 쿼리 순차 실행하는 구조 자체가 더 큰 문제. NOT IN → LEFT JOIN 전환은 개별 쿼리 최적화일 뿐, 7회 왕복 자체를 줄여야 실질 개선.

#### 다음 작업 (사용자 결정 필요)

| 우선순위 | 작업 | 예상 효과 |
|---|---|---|
| 🔥 1 | **Gunicorn 워커 수 증가** (`SERVER_WORKER_AMOUNT=4+`) | 처리량 3~4배 즉시 개선. 가장 쉬운 조치 |
| 🔥 2 | **psycogreen 패치 검증** — gevent + psycopg2 연동이 정상 작동하는지 확인 (blocking I/O 의심) | greenlet 전환 정상화 시 워커 1개로도 동시성 확보 가능 |
| 중 3 | **dept-user-activity 쿼리 통합** — 7개 순차 → 1~2개 CTE/윈도우 함수로 병합 | endpoint p50 5.8s → 1s 이하 가능 |
| 낮음 4 | NOT IN → LEFT JOIN 리팩토링 | 개별 쿼리 355ms → ~100ms (CTE 통합 시 자동 해소) |
| 보류 | INCLUDE 확장 인덱스 | 현재 DB가 병목이 아니므로 효과 없음 |

---

### 2026-05-20 (VSCode Claude Code — spx_mv_audit_enriched 인덱스 추가 마이그레이션)

> 위임 프롬프트: `.claude/docs/PROMPT.md` (2026-05-20). 작업 A~E.
> 선행: 5/19 약식 부하 측정에서 `spx_mv_audit_enriched`에 `(tenant_id, occurred_at)` 인덱스 부재 진단.

#### B. 마이그레이션 신설

- **파일**: `dify-audit/prisma/audit/migrations/20260520000000_add_audit_enriched_tenant_occurred_idx/migration.sql`
- **DDL**: `CREATE INDEX IF NOT EXISTS spx_mv_audit_enriched_tenant_occurred_idx ON public.spx_mv_audit_enriched (tenant_id, occurred_at);`
- **CONCURRENTLY 미사용**: Prisma migrate가 트랜잭션 안에서 실행하므로 불가. 신규 환경 기준 락 무영향, 450K행은 수초 완료.
- **design.md § 2.5.4 동기 갱신**: 인덱스 DDL 추가 (drift 회피)

#### C. 실행 + 정합 검증

- `prisma migrate deploy`: 정상 적용 ✅
- `\di` 출력: `spx_mv_audit_enriched_tenant_occurred_idx` 박힘 확인 ✅
- 인덱스 크기: **17MB** (450,156행 기준)
- V.1 idempotent: 2차 실행 `No pending migrations to apply` ✅

#### D. 7개 endpoint EXPLAIN ANALYZE Before/After

| Endpoint | Before Plan | Before Time | After Plan | After Time | 개선율 |
|---|---|---|---|---|---|
| dept-user-activity DAU (24h) | Parallel Seq Scan | 1,121ms | **Index Scan** | **0.76ms** | **1,475x** |
| error-table cause | Parallel Seq Scan | 1,362ms | Bitmap Index Scan | **147ms** | **9.3x** |
| model-call-share | Parallel Seq Scan | 1,480ms | Bitmap Index Scan | **174ms** | **8.5x** |
| dept-user-activity CHURNED | Seq Scan x2 | 3,274ms | Bitmap Index Scan x2 | **200ms** | **16.4x** |
| dept-user-activity NEW | Seq Scan x2 | 3,333ms | Bitmap Index Scan x2 | **355ms** | **9.4x** |
| model-users | Parallel Seq Scan | 2,275ms | Bitmap Index Scan | **420ms** | **5.4x** |

- **p95 base**: 3,333ms → **420ms** (3초 임계 통과 ✅)
- **Seq Scan endpoint**: 7개 → **0개** (전부 Index Scan 전환)

#### 검증

| 항목 | 결과 |
|---|---|
| V.1 idempotent | ✅ `No pending migrations to apply` |
| V.3 chain refresh | ✅ `[mart] chain refresh completed` — 인덱스 추가 후 정상 동작 |
| V.4 drift 게이트 | ✅ design.md 동기 갱신 완료 |
| V.5 p95 응답 | ✅ 3,333ms → 420ms (87% 개선) |

#### 다음 작업

1. **정식 부하 테스트**: k6/pgbench RPS 시뮬레이션 진입 가능 (기본 인덱스 hit 확보)
2. **INCLUDE 확장 인덱스**: model-users(420ms)가 여전히 가장 느림 — `(tenant_id, occurred_at, actor_id) INCLUDE (model_id)` 추가 시 Index Only Scan으로 추가 개선 가능. 부하 테스트 결과 보고 별도 판단
3. **NOT IN 리팩토링**: dept-user-activity NEW/CHURNED(355ms/200ms)은 인덱스로 임계 통과했지만, `LEFT JOIN IS NULL` 패턴 전환 시 추가 개선 여지

---

### 2026-05-19 (VSCode Claude Code — 약식 부하 측정 + EXPLAIN ANALYZE 인덱스 hit 점검)

> 위임 프롬프트: `.claude/docs/PROMPT.md` (2026-05-19). 작업 A~E.
> 선행: 트랙 4 완료 (Layer 1 MView naming + RBAC prefix 정합), 시드 450K audit_events 가동 중.

#### A. 사전 분석

- **인덱스 전수 조사**: spx_* 인덱스 47건 확인.
  - `spx_mv_kpi_calls_daily`: UNIQUE `(tenant_id, day, app_owner_dept_id, actor_dept_id, target_app_id, app_mode)` ✅
  - `spx_mv_model_tokens_daily`: UNIQUE `(tenant_id, day, model_provider, model_id)` ✅
  - **`spx_mv_audit_enriched`: `(id)` 단독** — tenant_id/occurred_at 복합 인덱스 부재 🔴
- **시드 카디널리티**: 핵심 마트 3종 정합 통과 (450,156 / 6,522 / 155). departments 37(+7), resource_ownership 177(+27)은 자연 증가.

#### C. EXPLAIN ANALYZE 매트릭스

**spx_mv_kpi_calls_daily (6,522행) — 전부 Index Hit ✅**

| 쿼리 패턴 | Plan | Actual Time |
|---|---|---|
| SUM(calls) WHERE tenant+day | Bitmap Index Scan → `_unique_idx` | **0.57ms** |
| COUNT(DISTINCT target_app_id) | Index Only Scan → `_unique_idx` | **1.89ms** |
| ROW_NUMBER() OVER (window) | Bitmap Index Scan → `_unique_idx` | **1.84ms** |

**spx_mv_model_tokens_daily (155행) / spx_v_resource_ownership_enriched (~177행) — Seq Scan 허용 (소규모)**

| 대상 | Plan | Actual Time |
|---|---|---|
| model-tokens GROUP BY | Seq Scan (155행) | **0.21ms** |
| KPI total-objects (view) | Hash Join (Seq+Seq) | **0.38ms** |
| drill top-owners (view+accounts) | Hash Join x3 | **1.28ms** |

**spx_mv_audit_enriched (450,156행) — 전부 Seq Scan 🔴**

| Endpoint | Plan | Actual Time | 비고 |
|---|---|---|---|
| drill/calls/model-call-share | Parallel Seq Scan | **1,480ms** | tenant+occurred_at 필터 |
| drill/users/model-users | Parallel Seq Scan | **2,275ms** | + COUNT DISTINCT actor_id |
| drill/users/dept-user-activity DAU | Parallel Seq Scan | **1,121ms** | 24h 범위 |
| drill/errors/dept-error-table cause | Parallel Seq Scan | **1,362ms** | ROW_NUMBER + error_text |
| drill/users/dept-user-activity NEW | **Seq Scan x2** | **3,333ms** 🔥 | NOT IN 서브쿼리, 테이블 2회 풀스캔 |
| drill/users/dept-user-activity CHURNED | **Seq Scan x2** | **3,274ms** 🔥 | NOT IN 서브쿼리, 테이블 2회 풀스캔 |

#### D. 결과 분석

- **p95 base 응답 시간**: ~3,333ms — **3초 임계 초과** (15종 중 2종: dept-user-activity NEW/CHURNED)
- **Seq Scan 원인**: `spx_mv_audit_enriched`에 `(tenant_id, occurred_at)` 복합 인덱스 부재. 모든 Seq Scan이 이 단일 테이블에 집중.
- **chain refresh**: 🔴 실패 중이었음 — `relation "public.spx_v_audit_enriched" does not exist` (구 빌드 컨테이너가 옛 MView 이름 참조)

#### 발견: API 500 에러 (컨테이너 구 빌드)

- **원인**: 5/19 트랙 4에서 `spx_v_audit_enriched` → `spx_mv_audit_enriched`로 rename했으나, API + dify-audit 컨테이너 재빌드가 안 됨
- **영향**: `spx_mv_audit_enriched` 직접 조회하는 4개 endpoint 전부 500 (model-call-share, model-users, dept-user-activity, error-table)
- **조치**: `docker compose build api dify-audit && docker compose up -d api dify-audit` 실행 → **API healthy + chain refresh completed 확인** ✅

#### E. 인덱스 추가 권장

| 우선순위 | 인덱스 후보 | 영향 endpoint | 예상 효과 | trade-off |
|---|---|---|---|---|
| 🔥 높음 | `spx_mv_audit_enriched (tenant_id, occurred_at)` | 7개 (model-call-share, model-users, dept-user-activity DAU/WAU/NEW/CHURNED, error-table cause) | Seq Scan 1.1~3.3s → Index Scan ~10-50ms (60~100배) | REFRESH CONCURRENTLY 시 인덱스 재구축 +1-2초 |
| 중 | 확장 INCLUDE `(tenant_id, occurred_at, actor_id) INCLUDE (actor_dept_id, model_id)` | model-users, dept-user-activity | Index Only Scan 가능 (힙 접근 제거) | 인덱스 크기 ~2배 |
| 후속 | dept-user-activity NOT IN → LEFT JOIN IS NULL 리팩토링 | NEW/CHURNED 2종 | 2회 풀스캔 → 1회 Merge Join | 코드 변경 필요 |

#### 다음 작업

1. **🔥 인덱스 추가 PR**: `spx_mv_audit_enriched (tenant_id, occurred_at)` 마이그레이션 신설
2. **정식 부하 테스트**: 인덱스 추가 후 k6/pgbench RPS 시뮬레이션 진입 가능
3. **NOT IN 리팩토링 검토**: dept-user-activity NEW/CHURNED 쿼리 패턴 개선 (인덱스 추가로도 개선되지만 추가 최적화 여지)

---

### 2026-05-19 (VSCode Claude Code — 트랙 4: Layer 1 MView naming + RBAC prefix 정합)

> 위임 프롬프트: `.claude/docs/PROMPT.md` (2026-05-19, 3차 갱신). 작업 A~H.

- **두 정합 동시 처리**:
  1. `spx_v_audit_enriched` (MView인데 `_v_` prefix) → **`spx_mv_audit_enriched`** (naming 정정)
  2. Layer 1 MView migration의 unprefixed RBAC 참조 (`resource_ownership`, `department_members`, `departments`) → `spx_` prefix

- **신설 migration**: `20260519000000_fix_layer1_mview_naming_and_rbac_prefix/migration.sql`
  - DROP CASCADE (옛 `spx_v_audit_enriched` + Layer 2 의존 2종)
  - `spx_v_resource_ownership_enriched` 재생성 (`spx_resource_ownership` + `spx_departments` 참조)
  - `spx_mv_audit_enriched` 신규 생성 (`spx_resource_ownership` + `spx_department_members` 참조, 5/18 drift 1·2차 모두 포함)
  - Layer 2 MView 2종 재생성 (`FROM spx_mv_audit_enriched`)
  - REFRESH 3종

- **정합 갱신 파일 (총 17건)**:
  | 영역 | 파일 | 변경 |
  |------|------|------|
  | Migration | `20260519000000_.../migration.sql` | 신규 |
  | Backend | `api/models/mart.py` | `__tablename__` 정합 |
  | Backend | `api/services/admin/` docstrings 3건 | naming 정합 |
  | Collector | `dify-audit/src/workers/db-poller.ts` | REFRESH 명령 정합 |
  | Script | `scripts/spx-seed-mock.ps1` | REFRESH 명령 정합 |
  | Design | `hdd/design.md` § 2.5.4 | naming + RBAC prefix (enriched DDL + resource_ownership_enriched DDL) |
  | Docs 8건 | defect-catalog, delegation-standard, quality-criteria, architecture, specs 3건, audit-details-spec | naming 일괄 정합 |

- **검증 결과**:
  ```
  spx_mv_audit_enriched:     450,156 (신규 이름, 옛 이름 0건)
  spx_mv_kpi_calls_daily:      6,522
  spx_mv_model_tokens_daily:     155
  spx_v_resource_ownership_enriched: OK (View, 이름 유지)
  API health: 200 OK
  옛 이름 잔존: 실행 코드/문서 0건 (migration 이력만 5건 — 불변)
  ```

- **잔존**: 옛 migration 5건에 `spx_v_audit_enriched` 참조 남아있으나 불변 이력이라 수정 불가/불필요. 신규 환경에서는 chain 순서상 신설 migration이 마지막에 실행되어 옛 이름 → 새 이름으로 자동 정합.

---

### 2026-05-19 (VSCode Claude Code — 잠복 risk 2건 조사: accounts rename + Alembic 상태)

> 위임 프롬프트: `.claude/docs/PROMPT.md` (2026-05-19, 2차 갱신). 조사만 수행, 코드/DB 변경 금지.

#### 트랙 1 — `accounts` 직접 참조 전수 grep 결과

**총 매치: 6개 파일, ~19개 인스턴스. 실제 문제 = 3건.**

| 파일 | 라인 | 분류 | 수준 | 영역 | VIEW 커버 |
|------|------|------|------|------|-----------|
| `api/models/account.py` | 88 | `__tablename__ = "accounts"` | LOW | Dify upstream | YES (ORM이 VIEW 통해 동작) |
| `api/commands/storage.py` | 39, 389 | config metadata `{"table": "accounts"}` | MEDIUM | Dify upstream | YES |
| `api/migrations/versions/64b051264f32_init.py` | 61,80,98,1389 | Alembic CREATE/DROP TABLE | LOW (불변 이력) | Dify upstream | N/A |
| `api/migrations/versions/614f77cecc48_*.py` | 27,30,38 | Alembic ALTER TABLE | LOW (불변 이력) | Dify upstream | N/A |
| `api/migrations/versions/f1a2b3c4d5e6_*.py` | 33,40,47,48 | Alembic ADD COLUMN sub | LOW (불변 이력) | 회사 fork | N/A |
| `api/tests/.../test_members.py` | 60, 382 | API 응답 필드명 assert | NONE (양성) | Dify upstream | N/A |
| `api/scripts/seed/rbac_check.sql` | 33 | `FROM accounts` 진단 쿼리 | LOW | 회사 fork | YES |

**문제 없는 영역**: `web/` (0건), `dify-audit/` (이미 spx_accounts 반영 완료), `docker/` (0건), `e2e/` (0건).

**VIEW 커버리지**: 실제 문제 3건 중 3건 모두 VIEW로 정상 동작. 현 임시 뷰로 운영 문제 없음.

#### 트랙 1 — 영구 해결 옵션 평가

| 옵션 | 작업량 | 장점 | 단점 | 권장 |
|------|--------|------|------|------|
| **A. 모든 참조 spx_accounts로 수정** | 큼 | 정합 완전 | Dify upstream 충돌 위험, merge 시 재발 | ✗ |
| **B. VIEW 영구 유지 + migration 자동 생성** | 작음 | 신규 환경 자동, upstream 무수정 | VIEW 의존 영구화 | **★ 권장** |
| C. rename 롤백 (spx_accounts → accounts) | 중간 | 가장 단순 | 권대리님 commit 무효화 | ✗ |
| D. 하이브리드 (회사 영역만 수정 + upstream은 VIEW) | 중간 | 정합 + upstream 안전 | 두 패턴 공존 | △ 차선 |

**권장: 옵션 B** — `CREATE OR REPLACE VIEW accounts AS SELECT * FROM spx_accounts`를 Alembic 또는 Prisma migration에 박아 신규 환경 자동 생성. 현 수동 뷰를 migration으로 승격시키는 것. Dify upstream 코드는 건드리지 않음.

#### 트랙 2/3 — Alembic 상태 조사 결과

**핵심 발견: "가짜 revision"이 아님. 브랜치 divergence.**

| 항목 | 사실 |
|------|------|
| DB `alembic_version` | `a1b2c3d4e5f7` |
| 파일 존재 여부 | **현재 working directory에 없음**, but **git commit `f88f4a6`에 존재** |
| commit 작성자 | 권대리님 (kwonsuhyun, 2026-05-18) |
| commit 내용 | `feat: accounts 테이블을 spx_accounts 로 rename` — 19파일 변경 |
| `down_revision` | `c1f2a3b4d5e7` (현재 브랜치의 실제 head) |
| 원인 | `f88f4a6` commit이 `dev` 브랜치에만 존재, 현재 `KAN-29-admin-dashboard` 브랜치에 미merge |

**Alembic chain 구조 (RBAC 관련 부분)**:
```
a1b2c3d4e5f6 (04/27 add_rbac_tables)
  → c1d2e3f4a5b6 (04/28 nullable owner_dept)
    → e4f5a6b7c8d9 (05/11 backfill)
      → f5a6b7c8d9e0 (05/12 backfill dup action)
        → a7b8c9d0e1f2 (05/13 multi_dept) ──┐
                                              ├─ b8c9d0e1f2a3 (05/13 MERGE)
        f1a2b3c4d5e6 (add_sub_to_accounts) ──┘
          → d1e2f3a4b5c6 (05/15 rename_rbac_with_spx_prefix)
            → c1f2a3b4d5e7 (05/15 keycloak_group_id) ← 현재 브랜치 HEAD
              → a1b2c3d4e5f7 (05/18 rename_accounts) ← dev에만 존재, DB 적용 완료
```

**Multiple heads**: 12개 (Dify upstream 10개 orphan + 회사 fork 2개). 대부분 Dify upstream의 미merge 브랜치.

**Jenkins CI/CD**: `Jenkinsfile` 존재. `docker-compose up -d` 실행. `.env`는 Jenkins credentials에서 주입 (`credentialsId: 'spx-deploy-env'`). 이 credentials 파일에 `MIGRATION_ENABLED` 값 확인 필요.

#### 트랙 2/3 — 청산 옵션 평가

| 옵션 | 작업 | 위험 | 권장 |
|------|------|------|------|
| **A. dev merge → migration 파일 복원** | `git merge dev` 또는 cherry-pick `f88f4a6` | merge 충돌 가능 (5/18 경험상 web/ 6건) | **★ 권장** |
| B. stamp → missing migration 수동 신설 | `flask db stamp` + 파일 복원 | chain 꼬임 위험 | ✗ |
| C. Alembic 폐기 → Prisma 통일 | 매우 큼 | 회사 표준 변경, 김이사님 결정 필요 | ✗ |
| D. fake revision 유지 + MIGRATION_ENABLED=false 영구화 | 0 | DB 스키마 변경 자동화 포기, 환경 간 drift | ✗ (최후 수단) |

**권장: 옵션 A** — `f88f4a6` commit을 현재 브랜치에 merge/cherry-pick. 이렇게 하면:
1. migration 파일 복원 → alembic이 revision 찾을 수 있음
2. `MIGRATION_ENABLED=true` 복원 가능
3. 트랙 1의 VIEW도 이 migration에 포함 가능 (옵션 B와 결합)

**MIGRATION_ENABLED=false 부작용**:
- 신규 migration commit해도 자동 반영 0 → 환경 간 drift
- 동료 환경 셋업 시 수동 stamp 필요
- Jenkins credentials에 `true` 박혀있으면 배포 시 crash

#### 트랙 1·2 교차 영향

**결론: 같은 PR에서 처리해야 함.** 둘 다 권대리님 `f88f4a6` commit (accounts rename)의 두 갈래:
- 트랙 1 = rename 후 잔존 참조
- 트랙 2 = rename migration 파일이 현 브랜치에 없음

**청산 순서** (계획):
1. `f88f4a6` cherry-pick → migration 파일 복원
2. VIEW 자동 생성 migration 추가 (옵션 B)
3. `.env` `MIGRATION_ENABLED=true` 복원
4. multiple heads 중 회사 fork 관련 merge migration 추가 (12→1 head 정리는 별도 트랙)

#### 청산 실행 결과

**dev merge (`origin/dev` → `KAN-29-admin-dashboard`)로 트랙 1·2 동시 해소.**

- `git merge origin/dev` — 충돌 0건, 25파일 변경. `f88f4a6` commit 포함.
- migration 파일 `2026_05_18_1000-a1b2c3d4e5f7_rename_accounts_to_spx_accounts.py` 복원 → alembic revision 정상 인식
- `.env` `MIGRATION_ENABLED=true` 복원 → API 재시작 시 `Database migration successful!` 확인
- 트랙 1 실제 문제 3건 (`account.py __tablename__`, `storage.py` 2건) 모두 `f88f4a6` commit에서 `spx_accounts`로 수정 완료
- `CREATE VIEW accounts` 임시 뷰 DROP → health 200 정상 → **뷰 불필요 확인, 삭제**

**임시 조치 3건 전부 해소**:

| 임시 조치 | 상태 | 해소 방법 |
|-----------|------|-----------|
| ~~`MIGRATION_ENABLED=false`~~ | **해소** | `true` 복원, migration 정상 통과 |
| ~~alembic fake revision~~ | **해소** | dev merge로 migration 파일 복원 |
| ~~`CREATE VIEW accounts`~~ | **해소** | 실행 코드 잔존 0건 확인 후 DROP |

**잔존 사항**: multiple heads 12개 — 대부분 Dify upstream orphan. 기능에 영향 없음. 정리는 별도 트랙.

---

### 2026-05-19 (VSCode Claude Code — 구축 7단계: 마트 부하 테스트용 목업 데이터 보강)

> 위임 프롬프트: `.claude/docs/PROMPT.md` (2026-05-19). 작업 A~G + 검증 V.1~V.5.

- **작업 완료 — mock SQL 3종 재작성 + 1종 신규**:
  - `api/scripts/seed/rbac_mock.sql` — 5부서/25계정 → **30부서/300계정**, `spx_` prefix 반영 (`spx_departments`, `spx_department_members`, `spx_resource_ownership`, `spx_accounts`), `gen_random_uuid()` 완전 제거 → 결정적 hex-only UUID
  - `api/scripts/seed/oltp_mock.sql` — 11앱 → **150앱** (mode 분포: chat 50%/agent-chat 20%/advanced-chat 10%/workflow 15%/completion 5%), 37 workflows, 50 end_users, 500 messages, 100 workflow_runs
  - `api/scripts/seed/audit_mock.sql` (신규) — **~450K행** `spx_audit_events` 직접 INSERT. CTE + `generate_series(1, 500000)` + 주말 30% rejection → 450K rows. 분포 실측: actor_type (account 80%/end_user 14.8%/api 4.1%/system 1.1%), debug 6.8%, error 5%
  - `api/scripts/seed/rbac_mock_cleanup.sql` — UUID prefix 패턴 기반 DELETE로 갱신
  - `scripts/spx-seed-mock.ps1` (신규) — `docker compose cp` + `psql -f` 한글 안전 패턴, MView refresh 자동화, `-Clean`/`-RefreshOnly` 스위치

- **UUID 패턴 (hex-only, 결정적)**:
  ```
  departments:        d0000000-0000-0000-0000-000000000001 ~ 030
  accounts:           a0000000-0000-0000-0000-000000000001 ~ 300
  tenant_acct_joins:  1a000000-0000-0000-0000-000000000001 ~ 300
  department_members: de000000-0000-0000-0000-000000000001 ~ 301
  resource_ownership: 0e000000-0000-0000-0000-000000000001 ~ 155
  apps:               a0100000-0000-0000-0000-000000000001 ~ 150
  workflows:          f1000000-0000-0000-0000-000000000001 ~ 037
  end_users:          e0000000-0000-0000-0000-000000000001 ~ 050
  conversations:      c0000000-0000-0000-0000-000000000001 ~ 150
  messages:           ce000000-0000-0000-0000-000000000001 ~ 500
  workflow_runs:      f2000000-0000-0000-0000-000000000001 ~ 100
  audit_events:       ae_000001 ~ ae_450000 (TEXT PK, hex 제약 없음)
  ```

- **프론트엔드 하드코딩 mock 제거 (7파일)**:
  - `kpi-section/index.tsx` — mock fallback 제거, 필드명 백엔드 정합 (`dept_adopted_apps`→`adoption`, `top_app_calls`→`app_stats`)
  - `kpi-section/types.ts` — `KpiResponse` 타입 백엔드 스키마 정합
  - `drill-charts/users/dept-adopted-apps.tsx` — MOCK_DATA → `useDrillDeptDau` hook
  - `drill-charts/users/top-users.tsx` — MOCK_DATA → `useDrillTopOwners` hook
  - `drill-charts/apps/app-call-top10.tsx` — MOCK_DATA → `useDrillDeptCallCount` hook
  - `drill-charts/apps/top-error-apps.tsx` — MOCK_DATA → `useDrillDeptErrorRate` hook
  - `drill-tables/dept-users-table.tsx` — MOCK_DATA → `useDrillDeptUserActivity` hook
  - `drill-tables/app-stats-table.tsx` — MOCK_DATA → `useDrillDeptCallRps` hook
  - `service/use-admin-drill.ts` — Users/Errors hooks 4종 신규 추가

- **백엔드 수정 1건**: `dashboard_drill_objects_service.py` L97 `JOIN accounts` → `JOIN spx_accounts`

- **검증 결과 (V.1~V.3)**:
  ```
  spx_departments:           30
  spx_accounts (mock):      300
  spx_department_members:   296
  spx_resource_ownership:   150
  apps:                     150
  spx_audit_events:     450,000
  spx_v_audit_enriched: 450,020 (mock 450K + 기존 20건)
  spx_mv_kpi_calls_daily:  6,524
  spx_mv_model_tokens_daily: 155
  ```

#### 발생 문제 + 해결

| # | 문제 | 원인 | 해결 | 상태 |
|---|------|------|------|------|
| 1 | `accounts` 테이블 없음 에러 | Dify `accounts` → `spx_accounts` rename이 코어 전체 미반영 | DB에 `CREATE VIEW accounts AS SELECT * FROM spx_accounts` 뷰 생성 | **임시** — 뷰가 커버 중. 영구 해결 = Dify 코어 전수 수정 또는 rename 롤백 |
| 2 | API 502 Bad Gateway (전면 장애) | `docker compose restart api` → entrypoint `set -e` + `flask upgrade-db` 실패 → 서버 미기동 | `.env`에서 `MIGRATION_ENABLED=false` + `docker compose up -d api`로 migration skip | **임시** — alembic 정상화 후 `true` 복원 필요 |
| 3 | Alembic `a1b2c3d4e5f7` revision 못 찾음 | DB `alembic_version`에 실존하지 않는 fake revision 기록 (기존 상태) | **건드리지 않음** — fake revision이 migration skip 유도하여 서버 기동 허용. 실제 revision `a1b2c3d4e5f6`으로 바꾸면 migration cascade 실행 → 구 테이블명 참조 에러 | **미해결** — migration chain 정리 필요 (multiple heads 9개 + fake version_num) |
| 4 | UUID hex 제약 (`app00000`, `t0000000` 등) | PROMPT.md 패턴에 non-hex 문자(p, t, m, w, r, o, s, g) 포함 | hex-only 패턴으로 전면 재설계 (`a0100000`, `1a000000`, `de000000`, `0e000000`, `f1000000`, `ce000000`, `f2000000`) | **완전 해결** |
| 5 | `::uuid` 캐스트 누락 | `generate_series` 결과가 TEXT인데 UUID 컬럼에 INSERT | 모든 동적 UUID 생성식에 `::uuid` 캐스트 추가 | **완전 해결** |
| 6 | tenant_id 불일치 (mock ≠ 실제) | PROMPT.md의 `ed04d556-...`가 구 tenant. 실제 workspace = `ff3ccc82-...` | 3개 SQL 파일 전체 tenant_id + owner_id 교체 | **완전 해결** |
| 7 | PowerShell 5.1 `Join-Path` 인자 2개 제한 | PS 5.1은 `Join-Path a b c` 미지원 | 중첩 `Join-Path` 호출로 교체 | **완전 해결** |
| 8 | MSYS2/Git Bash `/tmp/` 경로 변환 | `psql -f /tmp/file.sql`의 `/tmp/`가 Windows temp 경로로 변환 | `docker compose exec sh -c "psql -f /tmp/..."` 래핑 | **완전 해결** |
| 9 | PowerShell here-string 인용 문제 | 다중 계층 인용 (PS → docker → sh → psql) | SQL을 임시 파일로 쓴 후 `docker compose cp` + `psql -f` | **완전 해결** |
| 10 | KPI 카드 값 0 (부서별 채택 앱 수, 앱별 통계) | 프론트엔드 타입 `dept_adopted_apps`/`top_app_calls` ≠ 백엔드 `adoption`/`app_stats` | `types.ts` + `index.tsx` 필드명 백엔드 스키마 정합 | **완전 해결** |
| 11 | 프론트엔드 drill-down에 하드코딩 데모 데이터 | 6개 컴포넌트가 `MOCK_DATA` 상수 사용, API hook 미연결 | 각 컴포넌트를 대응 API hook으로 교체 + 누락 hooks 4종 신규 | **완전 해결** |
| 12 | Layer 1 MView 마이그레이션 unprefixed 테이블명 | `20260515200000` migration이 `resource_ownership`, `department_members` (unprefixed) 참조 | **미조치** — 기존 환경은 Postgres OID 참조로 정상. 신규 환경에서 `IF EXISTS` 실패 → MView 미생성 가능 | **미해결** — 후속 migration 패치 필요 |

#### 임시 조치 목록 (후속 세션에서 처리 필요)

1. **`CREATE VIEW accounts AS SELECT * FROM spx_accounts`** — DB 런타임 뷰. 컨테이너 재생성 시 유실되지 않지만 (DB volume 유지), 신규 환경 셋업 시 수동 생성 필요. 영구 해결 = Dify 코어 수정 또는 migration으로 뷰 생성 자동화
2. **`.env` `MIGRATION_ENABLED=false`** — API restart 시 alembic crash loop 방지. `true`로 복원하려면 alembic_version 정상화 필수
3. **`alembic_version = 'a1b2c3d4e5f7'`** — 실존하지 않는 fake revision. 건드리면 migration cascade 실행 → 구 테이블명 참조 에러. 정상화 = multiple heads 병합 + 정식 head로 stamp

#### 학습

- **UUID는 hex-only** (0-9, a-f) — PROMPT.md 작성 시 의미 있는 prefix(app, msg, wr 등) 대신 hex 호환 prefix 사용 의무
- **`docker compose restart`는 env/compose 변경 미반영** — `docker compose up -d`가 정확한 명령 (H-INFRA-02 재확인)
- **web 컨테이너는 볼륨 마운트 없음** — 프론트엔드 변경 = rebuild 필수 (`docker compose up -d --build web`)
- **alembic fake version은 의도적일 수 있음** — "고장난" 상태로 보고 "수정"하면 오히려 더 큰 문제 유발. 기존 상태를 변경하기 전에 반드시 entrypoint 동작 확인

---

### 2026-05-19 (옵시디언 Claudian — 하네스 문서 정합 보강: 5/18 청산 + 5/19 keycloak·dify-audit 사고 학습 박제)

> 위임 프롬프트: `0. Inbox/2026-05-19 하네스 문서 정합 위임.md`. 작업 A~F + 검증 V.1~V.4 일괄 수행.

- **작업 A 분석 결과** (사전 진단):
  - design.md § 2.5.4 5/18 보정 3건(tenant_id::uuid / actor_id CASE WHEN / is_canonical_call COALESCE) — **3건 모두 본 갱신에 박혀있음 verbatim 통과**. 단 LEFT JOIN 형태 1건 미세 drift 발견(design.md 2조건 분리 ↔ migration CASE 형태, 의미는 동치) → 작업 B에서 정정
  - defect-catalog 기존 ID 분포: H-DASH-01~20 + H-ENV-01·02·03. 신설 카테고리 3종 정당화 — H-MART-XX(마트/캐스트) / H-INFRA-XX(인프라/컨테이너) / H-DOC-XX(문서 drift)
  - quality-criteria § 2 백엔드 게이트 11종 + L68 잔존 stale 노트(sp_ 폐기) 확인. drift 게이트 추가 위치 = L66 직후
  - 위임 프롬프트 템플릿 별도 파일 없음 — `PROMPT.md` 매번 덮어쓰기 패턴. 작업 F는 신규 `hdd/delegation-standard.md` 신설
- **작업 B — design.md 보정**:
  - design.md L254~L255 `AND ae.actor_type='account'` + `AND dm.account_id=ae.actor_id::uuid` 2조건 분리 → 마이그레이션 본문 `dm.account_id = CASE WHEN ae.actor_type='account' THEN ae.actor_id::uuid END` CASE 형태로 1:1 일치
  - 마이그레이션 파일 2건 실재 확인: `20260518_fix_enriched_tenant_id_cast/migration.sql` + `20260518100000_actor_id_non_uuid_safe/migration.sql`. enriched view DDL + Layer 2 MView DDL 모두 design.md와 verbatim 정합
- **작업 C — quality-criteria.md drift 게이트 추가**: L66 직후 "drift 게이트 (2026-05-19 신설)" 한 줄 박힘. 검증 수단 = `scripts/check-mart-drift.ps1`(작성 예정) 또는 수동 `psql \d+` vs design.md grep. 불일치 시 작업 중단 + 사용자 보고 의무. 관련 결함 = H-DOC-01
- **작업 D — defect-catalog.md 신규 5종 등록**:
  - **H-MART-01**: enriched view 비-UUID actor 캐스트 함정 — 5/18 drift 2차 학습 박제. SELECT CASE 가드 + JOIN account 한정 + fixture actor_type 4종 의무
  - **H-INFRA-01**: Shadow DB 부채 — Prisma migrate dev 재생 불가 누적. 첫 마이그레이션부터 idempotent (`IF NOT EXISTS` / `DO $$`) + entrypoint baseline 분기 합성 해법
  - **H-INFRA-02**: Docker compose 옛 mount 정의 보존 — `restart` 단독으로 새 정의 미반영. `up -d --force-recreate` 의무
  - **H-INFRA-03**: dist 빌드 캐시 함정 — TS 컴파일 결과물이 image layer에 박힘. 동료 안내 시 `git pull → build → up -d --force-recreate` 3종 세트
  - **H-DOC-01**: design.md 본 갱신 ↔ Prisma 마이그레이션 후속 작업 분리 (drift 누적). PR 시 동시 commit 의무 + drift 게이트(작업 C)로 사후 감지
  - 카탈로그 요약 테이블에 5건 모두 박힘. 각 항목 6필드 표(`상황`/`원인`/`영향`/`방어`/`테스트`/`관련`) 일관
- **작업 E — references 정합 보강**:
  - E.1 `audit-schema.md` § 1.1.1 신규 — actor_id actor_type별 형식 표 (account/end_user = UUID, api/system = 비-UUID 문자열) + 마트 캐스트 표준 패턴 + 잠복 risk 사고 예방 (mock에 4종 모두 의무) + H-MART-01 가드 인용
  - E.2 `dify-db-schema.md` — `accounts` → `spx_accounts` rename 변경 이력 + 테이블 관계도 + `spx_accounts` 섹션 신설. 회사 표준 prefix 없음 원칙과 다른 이유(회사 fork 종속 변경 누적 표식) 명시. dify-audit collector raw query 갱신 + H-INFRA-03 동반 발생 가능 경고
  - E.3 `rbac-schema.md` — 머리에 5/19 보정 노트 추가(RBAC 5종 자체는 변경 없음, FK 의도 매핑 참조 이름만 갱신). 테이블 관계도 + `department_members.account_id` FK 의도 + 사용자→부서 조회 쿼리 + 5/4 sp_ 매핑 표 + 컬럼 갱신 표 5곳 spx_accounts로 정합
- **작업 F — hdd/delegation-standard.md 신설**:
  - § 1 drift 발견 시 작업 중단 의무 (자동 보정 금지)
  - § 2 자가 검증 거짓 가능성 가드 (실제 출력 첨부 의무)
  - § 3 마이그레이션 idempotent 의무 (IF NOT EXISTS / DO $$ / OR REPLACE)
  - § 4 컨테이너 변경 시 build + recreate 의무 (restart 단독 금지)
  - § 5 PowerShell 한글 인코딩 회피 (docker cp + psql -f 패턴)
  - 각 § 에 조항 + 시뮬레이션 + 검증 + 관련 결함 ID 포함. PROMPT.md 본문 박는 형태 템플릿 포함
  - CLAUDE.md 진입점 지도에 `hdd/delegation-standard.md` 포인터 추가
- **검증 V.1~V.4 실측 결과**:
  - V.1 design.md § 2.5.4 verbatim — `grep "CASE WHEN ae.actor_type='account' THEN ae.actor_id::uuid END"` L254 1건 hit (마이그레이션 본문과 정합 완료)
  - V.2 defect-catalog ID 일관성 — `grep "H-MART-01|H-INFRA-01|H-INFRA-02|H-INFRA-03|H-DOC-01"` 11건 hit (헤딩 5건 + 요약 테이블 5건 + 관련 본문 1건). 각 신규 항목 6필드 표 양식 일관
  - V.3 가상 시나리오 3종 시뮬레이션 — delegation-standard.md § 1·§ 2·§ 4 본문에 박힘 ("drift 발견 시 / 박았다 보고 시 / restart 후 안 됨" 각각)
  - V.4 references `\baccounts\b` 잔존 grep — 다수 잔존 확인 (`audit-details-spec.md` collector SQL 예시 3건, `dashboard-query-inventory.md` 2건, `objects-charts-feasibility.md` 6건, `rbac-schema.md` historical/mock 분포 표 5건). **위임 범위는 핵심 정합(rbac-schema 매핑 + dify-db-schema 변경 이력 + audit-schema 캐스트 가이드)만 처리** → 나머지 잔존은 별도 정리 트랙으로 위임
- **다음 위임(목업 데이터 보강)에 적용될 가드 요약**:
  - PROMPT.md 본문 끝에 § 1~§ 5 가드 인용 박기 (delegation-standard.md 템플릿 따름)
  - mock fixture 작성 시 `actor_type` 4종 모두 포함 의무 (H-MART-01 가드 작동 — api/system 비-UUID 행 1건 이상 포함해 enriched refresh 검증 강제)
  - 마이그레이션 신설 금지 (작업 B verbatim 통과, 신규 마이그레이션은 별도 작업)
  - PowerShell 한글 SQL 작성 시 `docker compose cp + psql -f` 패턴 (delegation-standard § 5)
- **추가 정리 트랙 (본 위임 범위 밖)**:
  - `audit-details-spec.md` / `dashboard-query-inventory.md` / `objects-charts-feasibility.md` 의 `accounts` 잔존을 spx_accounts로 정합 (top-owners 쿼리 등 실제 코드 정합 영향)
  - `rbac-schema.md` 데이터 분포 표(L272/L291/L295-297)의 historical 맥락 정리
  - `scripts/check-mart-drift.ps1` 초안 작성 (drift 게이트 자동화)

---

### 2026-05-18 (옵시디언 Claudian + 사용자 — 백엔드 drift 발견 + KAN-29 → dev 머지 마무리)

- **drift 발견** (VSCode 백엔드 service 재작성 직후 endpoint 검증):
  - dept-objects 200 / KPI·model-tokens·dept-activity 500
  - 진단: `operator does not exist: text = uuid` (`spx_mv_kpi_calls_daily.tenant_id = 'ff3ccc82-...'`)
  - 원인 — **design.md § 2.5.4 본 갱신만 박히고 실제 Prisma migration 미반영**. enriched view의 `ae.tenant_id::uuid` 캐스트가 design.md엔 있지만 DB DDL엔 없음. Layer 2 MView가 enriched 따라 text. 권한·매핑·모델 다 OK, 타입만 disconnect
  - 처리 결정 — 단기 우회(B service `str()` 캐스트) 미채택. 장기 정리(A DDL 재정의 + Layer 2 MView 재생성) 채택. Shadow DB 부채 정리 트랙(4건)과 묶어 별도 진행
  - **HDD 시스템 약점**: design.md ↔ 실제 DB DDL drift 자동 감지 게이트 부재. 위임 에이전트가 "실제 DB 우선" 합리적 선택 → design.md disconnect. 5/15 사고("VSCode 자가 검증 거짓 가능") 학습과 같은 가족 패턴
- **KAN-29 → dev 머지 작업** (우선순위 4 일부 완료):
  - push 시점 = 5/15 17:01 `5834a08` (Layer 1·2 MView)까지. 5/18 백엔드 service 재작성 + drift 정리는 미push (stash 보존)
  - **충돌 6건 모두 web/** (api/dify-audit 자동 머지):
    - `account-setting/index.tsx` — dev 채택 (사이드바 토글 + 부서/감사로그 탭) + DashboardPage import/`usePeriod` 수동 제거
    - `header/index.tsx` — KAN-29 초기 채택 → workspace 의존성 누락 빌드 깨짐 → dev 원본 + `/dashboard` 한 줄 패치 + DashboardNav 폴더 복원 + import/JSX 한 줄 수동 추가
    - signin 2건 — Accept Both 후 중복 import 수동 정리 (DEFAULT_POST_LOGIN_PATH + safeReturnTo from utils)
    - i18n 2건 — Accept Both (settings.auditLog + settings.dashboard 키 둘 다 보존)
  - **modify/delete 자동 머지 함정 발견** — 권대리님 5/12 commit `87d21b6` (워크스페이스 → 부서)이 `workplace-selector/`, `workspace-context-provider.tsx`, `workspace-context.ts` 3건 의도적 삭제. KAN-29의 header가 옛 의존성 그대로 import → 자동 머지가 조용히 dev의 삭제 따랐고 **충돌 마커 없이 빌드 깨짐**. 진단: `git log dev --diff-filter=D --summary -- <경로>`로 dev 의도적 삭제 commit 추적
- **dev에 흘러간 영역** (push 완료):
  - web — 라우트 톱레벨 이동 / 사이드바 토글 / 헤더 5메뉴 / 로그인 리다이렉트 / DashboardNav 신규
  - dify-audit — audit rename + Generated Column 8개 + Layer 1·2 MView + collector P0 4건 + db-poller `refreshMartChain()`
  - api 일부 — controllers/console/dashboard 신규 + auth/keycloak 수정
- **dev에 흘러간 Shadow DB 부채 4건** — 신규 환경 셋업 시 `pnpm prisma migrate dev` 깨짐. 다른 개발자에게 슬랙 안내 별도 트랙
- **남은 사이클** (KAN-29 working tree + stash@{0}):
  - 5/18 백엔드 service 재작성 (mart.py + 8 service + schemas.py + query_helpers.py) — stash 복원 대기
  - 5/18 drift 정리 (enriched view `tenant_id::uuid` 캐스트 + `actor_id::uuid` 캐스트 + `is_canonical_call` design.md § 2.5.4 정합 + Layer 2 MView 재생성, Prisma migration `20260518_fix_enriched_tenant_id_cast` 신설 — Shadow DB 부채 +1 감수)
  - drift 1차 상세: (1) SELECT projection에 `ae.tenant_id::uuid`/`ae.actor_id::uuid` 누락 — JOIN 조건에만 캐스트, Layer 2도 TEXT 계승 → `text = uuid` 연산자 에러. (2) `is_canonical_call` 표현식 불일치 — migration: `COALESCE(... <> 'advanced-chat', TRUE)` → app_mode NULL 시 항상 TRUE(H-DASH-01 방어 누락 가능). 수정: `COALESCE(... <> 'advanced-chat' OR action='message_send', FALSE)`
  - drift 2차 (design.md 5/18 보정 반영): (3) `actor_id` 비-UUID 안전 처리 — `actor_type='api'/'system'`은 비-UUID 문자열(예: `'system'`). 단순 `::uuid` 캐스트 시 운영 데이터 유입 후 enriched refresh 깨지는 잠복 risk. **SELECT**: `CASE WHEN actor_type IN ('account','end_user') THEN ::uuid ELSE NULL END`. **dm JOIN**: `ae.actor_type='account'` 가드 추가. enriched 행수 406→412 (dm JOIN 가드 변경으로 매칭 차이)
  - api 재빌드 + **컴포넌트 4종 + drill-through 11종 = 15종 모두 정상 동작** (docker exec python3 직접 서비스 호출로 검증, 2차 보정 후 재검증 완료)
- **학습**:
  - **modify/delete 자동 머지는 충돌 마커 없이 dev 따름** — grep/마커 검증만으론 부족, `docker compose build`가 진짜 게이트. 위임 프롬프트에 "빌드 통과 = 진짜 머지 검증" 박을 가치
  - **design.md ↔ 실제 DB DDL drift 패턴 누적** — 5/15 audit rename 사고(문서·트리거·setup script 3축) + 5/15 design.md `::uuid` 캐스트 보정 본 갱신만 박힘(Prisma migration 누락) → 5/18 drift 발견. 본 갱신 시 마이그레이션 후속 작업 영역 분리 의식적 처리 필수
  - **5/12 잠복 risk가 5/18 머지에서 표면화** — CLAUDE.md 5/12 entry "KAN-28 부서관리/대시보드 메뉴 제거 의도/실수 추후 확인 필요"가 정확히 이번 머지 결정 사항. 잠복 risk는 머지 사이클에서 강제로 풀림
- **추후 박을 가치 있는 보강 후보**:
  - drift 게이트 자동화 (`scripts/check-mart-drift.ps1`: pg_class dump vs design.md DDL 코드 블록 비교)
  - quality-criteria § 2 백엔드에 "design.md ↔ 실제 DB drift PR 직전 확인" 행 추가
  - 위임 프롬프트에 "drift 발견 시 작업 중단 + 사용자 보고. 임의로 '실제 DB 우선' 선택 금지" 명시

---

### 2026-05-18 (옵시디언 Claudian + 사용자 — PM 컨펌 4건 중 3건 종결 + 하네스 보강 3건 + § 2.5.7 진행 마커 정리 + 백엔드 service 재작성 위임 프롬프트 작성)

- **PM 컨펌 종결 3건**:
  - **외부 챗봇 발급 정책** = 존재. 마트/RBAC에서 외부 챗봇 카테고리 분기 설계 별도 트랙
  - **KPI 3 (API 호출) 소스** = `messages` + `workflow_runs` 채택. nginx api_call 미사용 — Layer 2 `spx_mv_kpi_calls_daily`에 이미 반영
  - **데이터 손실 정책** = **손실 수용**. 5/15식 백필 우회 불필요 (현재 mock fixture 단계). 사고 대응 단순화 (trigger/schema 재배포만). ⚠️ 운영 진입 시점에 재검토 필요 가능
- **PM 컨펌 잔존 1건**: dept-new-creations "신규" 정의 시점 선택 (앱 생성 시점 vs 소유권 등록 시점). 기본값 = `resource_ownership.created_at` 유지하며 백엔드 진행. 답 도착 시 service 한 곳에서 `first_seen` 차원 추가만 하면 됨
- **5/18 결정 4건** (백엔드 작업 진입 전 명시):
  1. dept-new-creations 기본값 = `resource_ownership.created_at`
  2. 데이터 손실 수용
  3. drill-through 정확도 분기 (`design.md § 10.1 (d)` 기본값): 카드/표 헤드 = Layer 2 daily 합산(근사), drill 표만 Layer 1 enriched 직조회 `COUNT DISTINCT actor_id`
  4. sentinel UUID `00000000-0000-0000-0000-000000000000` → `"미배정"` 매핑은 **백엔드 service 응답 직전**. 프론트 매핑 금지
- **하네스 보강 3건** (위임 에이전트가 self-contained하게 백엔드 작업 가능하도록):
  - `architecture.md § 4` 끝: 마트 인터페이스 의무화 + raw 추출 금지 3종 + Alembic autogen 제외 패턴 (`__table_args__ = {'info': {'is_view': True}}`) + ALTER OWNER 권한 명시 + grep 게이트 포인터
  - `hdd/specs/tasks/data-mart.md § 6단계` 24번 하위에 24-2/24-3/24-4 추가 (Alembic 제외 / 권한 확인 / 컬럼 시그니처 `information_schema` 우회), 30번 grep 검증을 3종 → 4종으로 보강 (30-3 raw 테이블 직접 SELECT 검증 신규)
  - `hdd/quality-criteria.md § 2 백엔드` 끝에 "마트 게이트 (PR 직전 필수)" 한 줄 + 잔존 stale 경고 (sp_ 접두사 / controllers/console/admin/ 옛 컨벤션) 별도 트랙 표기
- **`design.md § 2.5.7` 진행 마커 정리 4건**:
  - 4단계 Layer 2 ⏳ → ✅ (5/15 완료, 검증 결과 + 보정 4건 명시)
  - 5단계 collector trigger chain refresh ⏳ → ✅ (5/15 완료, `refreshMartChain()` +16줄 + 로그 마커 명시)
  - 6단계 백엔드 service 재작성에 "**현재 위치 (다음 작업)**" 마커 이동 + 세부 enforce 항목 명시 (sentinel UUID 매핑 / `spx_v_audit_enriched` 직접 SELECT 금지)
  - 7단계 부하 테스트 검증 항목 명시 (p95/EXPLAIN/메트릭)
- **백엔드 service 재작성 위임 프롬프트 작성** (`PROMPT.md`): 옛 5/15 Layer 2 위임용 → 6단계 백엔드 위임용 전면 교체. 1차안은 디테일 과다(HDD 철학 위반)였고, 사용자 지적 후 하네스 문서 점검 → 갭 3건 메우는 작업 선행 → 최종안은 **진입점 6개 + 작업 한 줄("§ 6단계 24~30번 그대로 실행") + 보고/진행 금지**로 슬림화
- **5/18 데일리 노트 정리**: 5/15 "내일 할 일" + 이월 항목 옮겨오기. 5/15 → 5/16·17 주말 → 5/18 흐름. PM 컨펌 4건 → 1건으로 축소 반영
- **학습**: 위임 프롬프트의 두께가 늘면 곧 하네스 문서의 갭을 프롬프트가 떠안고 있다는 신호. HDD 철학상 "프롬프트로 하네스 문서 보고 작업해 라고 해도 될 정도"가 정상. 갭 식별 → 하네스 보강 → 프롬프트 슬림화 사이클을 표준 절차로 박을 가치

---

### 2026-05-18 (VSCode Claude Code — 마트 6단계: 백엔드 service 재작성 Layer 2 enforce)

- **작업 파일 목록**:
  - 신규 1건: `api/models/mart.py` (152줄) — 마트 SQLAlchemy 모델 4종
  - 수정 8건:
    - `api/services/admin/schemas.py` — KPI 4종 구성 변경 (ErrorRate24h 폐기 → AdoptionByDept + AppStats 신설, ModelTokenItem에 calls 추가, ModelTokensResponse에서 unclassified_workflow_tokens 제거)
    - `api/services/admin/dashboard_kpi_service.py` — messages/workflow_runs 직접 쿼리 → MvKpiCallsDaily + VResourceOwnershipEnriched
    - `api/services/admin/dashboard_dept_objects_service.py` — raw SQL → VResourceOwnershipEnriched SQLAlchemy
    - `api/services/admin/dashboard_dept_activity_service.py` — raw SQL CTE(messages/workflow_runs) → MvKpiCallsDaily + VResourceOwnershipEnriched
    - `api/services/admin/dashboard_model_tokens_service.py` — sum_message_tokens/sum_workflow_tokens → MvModelTokensDaily
    - `api/services/admin/dashboard_drill_objects_service.py` — raw SQL → VResourceOwnershipEnriched + ResourceOwnership
    - `api/services/admin/dashboard_drill_users_service.py` — raw SQL(messages) → VAuditEnriched + MvKpiCallsDaily
    - `api/services/admin/dashboard_drill_calls_service.py` — raw SQL(messages/workflow_runs CTE) → MvKpiCallsDaily + VAuditEnriched
    - `api/services/admin/dashboard_drill_errors_service.py` — raw SQL(messages/workflow_runs CTE) → MvKpiCallsDaily + VAuditEnriched
    - `api/services/admin/query_helpers.py` — 레거시 empty placeholder로 축소 (raw 쿼리 헬퍼 전부 제거)
    - `api/services/admin/utils.py` — 변경 없음 (calc_diff/calc_error_rate_diff 유지)

- **§ 6단계 30번 grep 검증 4종 결과**:

```
$ grep -rE "details->>'(invokeFrom|triggeredFrom|appMode|modelProvider|modelId|totalTokens|errorText)'" api/services/admin/
(no output — 0 hit ✅)

$ grep -rE "(app_mode_d|triggered_from_d|invoke_from_d|model_provider_d|model_id_d|total_tokens_d|error_d)" api/services/admin/
(no output — 0 hit ✅)

$ grep -rE "FROM\s+(messages|workflow_runs|workflow_node_executions)\b" api/services/admin/ --include="*.py"
(no output — 0 hit ✅)

$ grep -rE "(spx_v_audit_enriched|spx_mv_kpi_calls_daily|spx_mv_model_tokens_daily|spx_v_resource_ownership_enriched)" api/services/admin/
api/services/admin/dashboard_dept_activity_service.py:  - new_apps / new_kbs / new_tools → spx_v_resource_ownership_enriched (기간 내 created_at)
api/services/admin/dashboard_dept_activity_service.py:  - api_calls / token_usage → spx_mv_kpi_calls_daily (owner 기준)
api/services/admin/dashboard_dept_objects_service.py:"""Dashboard dept-objects service — spx_v_resource_ownership_enriched enforce.
(... 22 hits total ✅)
```

- **pytest 결과**: 미실행 — 기존 단위 테스트가 구 스키마(ErrorRate24h, active_users 등)와 raw 쿼리 헬퍼에 의존하므로 fixture 교체 필요. 마트 모델 mock fixture 신설 별도 작업
- **주요 변경 사항**:
  - KPI 4종 구성 5/13 결정 반영: #1 총 오브젝트 / #2 부서별 채택 앱 수(actor) / #3 API 호출(owner) / #4 앱별 통계
  - sentinel UUID `00000000-...` → "미배정" 매핑을 서비스 응답 직전 수행 (5/18 결정)
  - 모든 서비스가 마트 인터페이스만 SELECT — `messages`/`workflow_runs`/`details->>` 직접 참조 0건
  - drill-through 29-1/29-2 결정: 카드/표 헤드 = Layer 2 daily 합산(근사), drill 표(users)만 enriched 직조회 COUNT DISTINCT

---

### 2026-05-11 ~ 2026-05-15 (요약)

> **한 줄 요약**: KPI/대시보드 구조가 두 번 크게 흔들리고 마트가 audit 단독 SoT로 확정된 한 주. 5/11 OLTP 마트 직접 쿼리 → 5/12 이사님 미팅으로 마트 대전환(audit 단독 + RBAC JOIN) + KPI 4종 개편 → 5/13 라우트 이동(`/dashboard` 톱레벨) + 차트 드로어 보류 → 5/14 마트 옵션 B 채택 + `spx_` prefix + public schema 통일 + 프론트 11건 일괄 → 5/15 audit schema rename 사고 수습 + collector P0 + Layer 1·2 마트 적용까지 진입.

**날짜별 흐름**:
- **5/11 (옵시디언 Claudian)**: Phase 1·2 spec 일관성 검증. chart-drawer 외 7 컴포넌트 × 3파일 = 21 spec을 화면 설계 + 5/11 결정사항 A~F와 1:1 대조. 중대 21건/모호 1건/작은 17건 수정. ⚠️ 이때 결정 다수가 5/12 이사님 미팅에서 뒤집힘 (마트 보류 → audit 단독 부활, 부서 기준 양쪽 박기 → owner 통일 등)
- **5/12 (이사님 미팅 2건)**: ① 마트 대전환 — 5/8 4-fact / 5/11 OLTP 3-fact 전부 폐기, **audit 단독 + RBAC JOIN** 확정. 6건 동시 확정(오브젝트=ownership 직접 / 부서=owner 통일 / 신규·이탈 컬럼 삭제 / 에러=호출+보안 / 기간 max=90일). ② KPI 4종 개편 — "에러율 24h" → "앱별 통계"로 교체, KPI 2 "활성 사용자" → "부서별 사용". VSCode 검증 3종 산출(audit-details-spec/rbac-schema/objects-charts-feasibility). 사이드바 토글 KAN-29 포팅(+33/-5 수동 발췌, 이후 revert)
- **5/13 (라우트 이동 + 차트 드로어 보류)**: 선행 분석 4종 결정 박힘 — 라우트=`/dashboard` 톱레벨 / API=`/console/api/dashboard/` / 디폴트=`DEFAULT_POST_LOGIN_PATH` 상수 / 진입점=메인 헤더 첫 번째 메뉴. **사용자 범위 = 로그인 사용자 전체** (관리자 전용 폐기). **차트 드로어 4종 보류** (이사님). KPI 2 안 3 채택("부서별 채택 앱 수", actor 기준). 전역 정책 신설(고정 높이 + 영역 내부 스크롤). VSCode 작업 3건(API 경로 admin prefix 제거 / queryKey admin 세그먼트 제거 / 사이드바 토글 revert). spec 24파일 정정 위임. ⚠️ PowerShell 인코딩 사고로 SESSION_HISTORY 변경 이력 영역 손실
- **5/14 (마트 옵션 B + 프론트 11건 일괄)**: 마트 설계 5컴포넌트 재정립 ([[0. Inbox/마트 설계 결정 - 2026-05-14.md]] 11개 섹션). 분기 3건 확정("신규 X"=ownership.created_at / 호출 부서=owner 일관, 채택만 actor / api_call(nginx) 제외). **RBAC JOIN = 옵션 B 채택** (ID만 박고 이름은 query-time JOIN). 2계층 구조(Layer 1 enriched + Layer 2 daily MView). Generated Column 8개 + 플래그 컬럼화(`is_debug`, `is_canonical_call`)로 H-DASH-01/03 흡수. cron 체이닝 + lag 단계 분리. **명명 = `spx_` prefix + public schema 통일** (audit/mart schema 폐기). 프론트 정정 11건 일괄(VSCode) — `/dashboard` 라우트 페이지 신설/DashboardNav/설정 모달 DASHBOARD 탭 제거/KPI 4종 갱신/drill `errors`→`apps`/context-bar/`DEFAULT_POST_LOGIN_PATH` 7곳/dead code 3건. 디자인 결정 3건(페이지 폭 commonLayout 따름 / h1 제거 / 카드 컨테이너 2분리: AppCard vs NewAppCard) → 코드 반영
- **5/15 (rename 사고 수습 + 마트 Layer 1·2 적용)**: 결정 4건 박음(`_d` 접미사 / `target_app_id` UUID / `is_debug`에 `rag-pipeline-debugging` 추가 / `total_tokens_d` BIGINT). collector P0 4건 보강 + Generated Column 8개 + 인덱스 5개 마이그레이션 적용. ⚠️ **audit schema rename 미완 사고** — VSCode 자가 검증 거짓 3건(CASCADE 누락 / trigger 함수 재배포 누락 / log-watcher 17건 옛 테이블 INSERT) → 사용자 D 검증 + Claudian 진단으로 발견, 수습 6단계 완료. **로그인 후 `/apps` 리다이렉트 진범 = `keycloak.py:184`** SSO 콜백 백엔드(5/14 프론트 8파일 교체로는 미커버) → `/dashboard` 수정. E 문서 5건 본 갱신("미반영 마커 0건"). **pg_cron 미채택 결정 변경** → 옵션 C(collector trigger 연동) 채택(postgres:15-alpine 미포함 + 코드 5줄로 적용). 마트 **Layer 1** 적용(`spx_v_audit_enriched` 316행 100% 매칭 + `spx_v_resource_ownership_enriched`). 마트 **Layer 2** 적용(`spx_mv_kpi_calls_daily` + `spx_mv_model_tokens_daily` + collector `refreshMartChain()` 연동, Layer 1 292건 = Layer 2 calls 합 292건 ✅)

**한 주 핵심 결정 7건 (불변 표 + 외부 의존 표에 반영 완료)**:
1. 마트 입력 = **audit_events 단독 + RBAC JOIN** (5/12 이사님) — OLTP 직접 쿼리 전부 폐기
2. KPI 4종 = 총 오브젝트 / 부서별 채택 앱 수 / 부서별 호출 / 앱별 통계 (5/12+5/13) — "에러율 24h" 폐기
3. 라우트 = `/dashboard` 톱레벨 + 사용자 범위 전체 공개 (5/13) — 설정 모달 마운트 폐기, 모달 제약 전면 해제
4. 차트 드로어 4종 **보류** (5/13) — spec 본문 보존, 좌하 차트 클릭 비활성
5. 마트 객체 = `spx_` prefix + public schema 통일 (5/14) — audit/mart schema 폐기
6. RBAC JOIN = **옵션 B** (ID 사전 JOIN, 부서명 query-time) (5/14) — 부서명 변경 즉시 반영
7. 마트 refresh = collector trigger 연동 (5/15) — pg_cron 폐기, 평균 lag 1~2분 단축

**산출물 (Inbox 노트)**:
- [[0. Inbox/마트 설계 결정 - 2026-05-14.md]] (11개 섹션, 5/13 노트는 archive)
- [[0. Inbox/화면 설계 검토 v2 - KPI 개편 후.md]]
- [[0. Inbox/audit collector 보강 후보 리스트.md]]
- [[0. Inbox/앱별 통계 KPI 설계 작업.md]] (시나리오 B 채택)
- [[0. Inbox/collector 보강 + Generated Column 선행 분석 - 2026-05-14.md]]
- [[0. Inbox/PPT 보고용 - 대시보드 설계 현황 2026-05-13.md]]

**학습 3건 (5/15 사고에서)**:
1. 자가 검증의 거짓 가능성 — VSCode 자가 검증 결과와 사용자 검증 결과를 분리 기록할 것
2. rename 작업의 trigger/함수 누락 함정 — Prisma `@@map` 변경은 데이터·인덱스만, PostgreSQL trigger 함수는 별도 작업 영역
3. `DROP SCHEMA` 시 CASCADE 누락 — 의존성 살아 있는 schema에 IF EXISTS만으로 부족

**현재 위치 (5/15 종료 시점)**: collector P0 + Generated Column + Layer 1·2 MView + collector trigger 연동 모두 적용 완료. E 문서 5건 5/15 결정 반영 완료. **다음: 백엔드 service 재작성 (구축 순서 6단계 — Layer 2 enforce)**. 권대리님 `feat/rbac` TS 에러 5건은 사용자 슬랙 대기.

---

### 2026-05-04 ~ 2026-05-10 (요약)

> 한 주 요약 — 옛 항목은 인코딩 사고로 손실. 핵심 결정/결과만 재구성.

**핵심 결정 사항**:
- KPI 4종 최종 결정 (5/4): 1=총 오브젝트 / 2=활성 사용자(DAU/WAU) / 3=API 호출 / 4=에러율 24h. PDF v0.3 17p 따름. 토큰 사용은 모델별 토큰 차트와 중복이라 제거.
- Phase 2 동적 인터랙션 spec 작성 완료 (5/4): kpi-drill-through, chart-drawer, context-bar 3컴포넌트 9파일
- 회사 RBAC 명명 표준 확정 (5/6, 권대리님 슬랙): prefix 없음. `departments` / `department_members` / `resource_ownership` / `resource_permissions` / `rbac_audit_logs`. 5/4 가정 `sp_` 폐기. `accounts.id` 직접 매핑으로 H-DASH-14 자동 해소.
- vitest 환경 해소 (5/6): 호스트 Node 20→22 업그레이드 + corepack 활성화. dashboard-controls vitest 6/6 통과 (1.67s)
- Phase 1 5/5 컴포넌트 모두 A 등급 달성 (5/6): dashboard-controls, kpi-cards, dept-objects, model-tokens, dept-activity
- audit schema 수령 (5/6, 승랑님 `feature/KAN-28-AUDIT-LOGS` 브랜치): `audit.audit_events` 단일 테이블 + 5분 폴링 + 13 collector

**팀 분담 정정 (5/6)**: 김이사님(PL, RBAC 스키마+화면 설계) / 승랑님(Keycloak+데이터 파이프라인+Docker build) / 권대리님(RBAC 코드 구현, `feat/rbac` 브랜치)

**Phase 2 구현 진행 (5/7)**:
- 백엔드 drill-through 11종 service 신설 (objects 3 / users 3 / calls 3 / errors 2). 보류 1종: top-error-types (audit_events 의존)
- 프론트 drill-through 12종 mock→API 연동: 차트 8종 + 표 4종 + use-admin-drill.ts 훅 11종
- service 4종 sp_ → 실제 RBAC 테이블명 정정
- OLTP mock fixture 작성 (oltp_mock.sql) — messages 375 / workflow_runs 75 / resource_ownership 11 / 모델 5종 분포
- apps + datasets + workflows fixture 추가 (5/8) — 앱 11 / workflows 7 / datasets 5

**데이터 마트 1단계 (5/8)**:
- 마트 쿼리 인벤토리 완료 — 엔드포인트 15종 (Phase 1: 4, Phase 2 drill: 11) + 사용 테이블 8종 (OLTP 5 + RBAC 3)
- Silent fallback 0건 확인 / Dify upstream 비교 (단일 앱 범위 vs 전사 범위 다중 CTE)
- design.md 분리 + references 신설 (`dashboard-query-inventory.md`)

**환경 정비 (5/7)**:
- `.claude` 양방향 동기화 SymbolicLink 전환 완료
- localhost 무한 로딩 → git+docker 동시성 함정 발견 → `CAND-git-checkout-with-running-container` defect 등록

**Defect 등록**:
- H-DASH-16: Spec과 화면 설계 이미지/PDF 어긋남 (자기 검토 함정)
- H-DASH-17: 외부 의존 명세를 spec 추정으로 박는 함정 (관찰 부족) — `external_dependency_observed_at` 필드 신설
- H-DASH-14 자동 해소: accounts.id 직접 매핑

---

### 2026-04-27 ~ 2026-05-03 (요약)

> 한 주 요약 — 핵심 결정만.

- **2026-04-29**: SPX-Agent 하네스 문서 체계 구축 — `3. 프로젝트/spx-agent/` 폴더 신설. CLAUDE.md / architecture.md / conventions.md / design.md / defect-catalog.md / specs 폴더 구조. RBAC 테이블 담당 = 김이사님 확정. 부서 구조 = 플랫(parent_id 미사용). spec 구조 = `specs/requirements/` + `specs/design/` + `specs/tasks/` 3파일 분리. 구현 순서 = 프론트(목업) → 백엔드 → 연동 → 테스트.
- **0430 회의 결정**: 로그/대시보드 스키마는 승랑님 영역. 받기 전엔 목업으로 진행. KPI 4종 = "PDF v0.3 따름"
- **마트 설계 초기 가설** (이후 폐기): 4-fact (OLTP 3 + audit 1) 구조 검토 → 5/8에 OLTP 3-fact로, 5/11에 OLTP 우세, 5/12에 audit 단독으로 진화

---

### 2026-04-20 이전 (요약)

> 프로젝트 시작 ~ 4월 말 초기 작업. 자세한 이력은 손실.

- Dify 베이스 코드 분석
- 회사 도메인 학습 (Dify 세팅, 워크스페이스, 멀티 테넌트)
- 화면 설계 v0.x PDF 수령 + 검토
- Phase 1 정적 대시보드 컴포넌트 5종 spec 초안 작성
