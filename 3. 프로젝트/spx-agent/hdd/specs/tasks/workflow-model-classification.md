---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: 모델 차원 차트 노드 기반 재분류 (모델별 토큰 + 모델별 호출 점유)
harness: [H-DASH-01, H-DASH-02, H-DASH-03, H-DASH-09, H-DASH-11, H-INFRA-01, H-INFRA-03, H-DOC-01, H-MART-01]
date: 2026-06-10
last_updated: 2026-06-10
---
# 워크플로우/챗플로우 모델 분류 (B안) — Tasks

> **목적**: 모델 차원 차트 2종(모델별 토큰 · 모델별 호출 점유)에서 워크플로우/챗플로우를 노드 기반으로 재분류 → 현재 '미분류'(=챗플로우)와 워크플로우 누락 해소.
> **근거 조사**: [[3. 프로젝트/spx-agent/references/workflow-model-classification.md]] (소스 검증 전문 · A안/B안)
> **범위 원칙**: 모델 차원 2종만 노드로. 부서/앱 차원·canonical 호출수는 message_send 유지 (H-DASH-01). 순서: collector → 마트 → 서비스 → 프론트 → 테스트.

## 시작 전 컨텍스트 (신규 세션 필독)

- **코드 repo 루트**: `C:\Users\Administrator\Projects\spx-agent` (베이스 `dev`). **이 문서의 모든 상대경로(`dify-audit/...`, `api/...`, `web/...`)는 이 루트 기준.** 볼트(`3. 프로젝트/spx-agent/`)에는 하네스 문서만 있고 실제 코드는 Projects 경로에 있음 — 혼동 주의.
- **브랜치**: 기존 `KAN-29-admin-dashboard`에서 작업.
- **착수 전 PM 확정 1건**: 0-3 마트 통합(UNION 재작성) — spec은 확정 기준이나 PM 사인만 받고 진행.
- **시드 구조 (2026-06-10 확인)**: `spx-seed-mock.ps1` → `rbac_mock.sql`/`oltp_mock.sql`/`audit_mock.sql` 순. **audit는 `audit_mock.sql`이 `spx_audit_events`에 직접 INSERT**(collector 경유 아님). 차트/마트 검증은 이 직접 시드로. **단 collector(Prisma) 검증은 OLTP→collector 경로라 `oltp_mock.sql`에 `workflow_node_executions` 추가 필요**(1.5단계 A에서 6케이스 추가 완료).
- **⚠️ DB 분리 (2026-06-10 실측)**: `DIFY_DATABASE_URL`은 **원격 DB(192.168.10.194:15432)**를 가리킴 — 로컬 `docker db_postgres(5432)`와 **별개**. **OLTP 시드(`workflow_node_executions`)는 원격 DB에, audit 직접 시드는 로컬 DB에** 넣어야 함. 모르면 "collector가 0 events" 한참 헤맴. collector는 `DIFY_DATABASE_URL`(원격)을 읽음.
- **먼저 읽을 것**: ① [[3. 프로젝트/spx-agent/references/workflow-model-classification.md]] (소스 검증 전문 — 왜 이렇게 하는지 근거 전부) ② [[3. 프로젝트/spx-agent/hdd/specs/tasks/data-mart.md]] (마이그레이션/마트/refresh chain 컨벤션) ③ `SESSION_HISTORY.md` (최신 진행 상태)
- **마이그레이션 도구**: Prisma Migrate raw SQL (`dify-audit/prisma/audit/migrations/`) — **Alembic 아님**. `CREATE INDEX CONCURRENTLY`는 트랜잭션 밖이라 migration 불가 → 별도 SQL 수동 실행. `audit_writer` role ALTER 권한 부재 가능 → superuser 직접 DDL (data-mart 5/15 학습).
- **⚠️ 검증 환경 한계**: 현재 DB는 **목업 데이터**. 따라서 token 합 일치(6단계 task 22)·오프로딩 빈도(0-4)는 **운영 DB 진입 전까지 구조/쿼리 검증까지만** 가능. 실데이터 정합은 운영 반영 후 별도 확인.
- **이미 확정된 사실**(재조사 불필요): `MvModelTokensDaily`에 `tokens`·`calls` 컬럼 둘 다 존재(`api/models/mart.py:132-133`, 2026-06-10 확인). 모델 보유 노드 3종 = `llm`/`question-classifier`/`parameter-extractor`. 챗플로우 message 모델=NULL & 토큰=노드합(reference §4).

## ⚠️ 기존 작업과의 충돌/중복 (2026-06-10 SESSION_HISTORY 대조 — 착수 전 필독)

> **B안 결론(불변)**: workflow/챗플로우를 노드의 **실제 모델로 차트에 넣는다**. 아래는 "그렇게 하려면 기존 코드의 어느 부분을 손대야 하나"를 짚는 출발 상태 메모일 뿐, 방향을 바꾸자는 게 아님.

1. **[기존 코드가 반대로 돼 있음] 5/20 "모델 미분류 수정"** — 현재 `get_model_call_share`엔 `WHERE model_id IS NOT NULL`이 **의도적으로** 박혀 있어 workflow(model NULL)가 차트에서 **빠진 상태**(미분류 막대 지우려고). **B안 task 15 = 이 필터를 걷어내고 노드 실제 모델로 채워 넣음**(= 차트에 등장). 그 필터가 실수가 아니라 일부러 넣은 거라 구현자가 멈칫할 수 있어 못 박는 것 — 지우는 게 B안이 맞음.
2. **[이미 완료] 6/10 작업3** — 프론트 `unclassified_workflow_tokens`/미분류 막대/`COLOR_UNCLASSIFIED` 제거됨(잔존: 주석 1줄). → **task 16은 사실상 완료**, "실제 모델 표시 확인"만 남음.
3. **[결정 뒤집음] 6/9 테스트 + H-DASH-02** — `test_dashboard_model_tokens_service`가 "workflow 배제 = 마트 Layer2 WHERE 구조적 처리"로 재작성돼 있음(현 합의 = 구조적 배제). B안은 이를 "노드로 재유입"으로 뒤집으므로 **해당 테스트 재작성 필요**(아래 6단계 task 26 신설). H-DASH-02 defect-catalog는 2026-06-10 B안 방향으로 이미 갱신됨.
4. **[이름 변경됨] 5/20 rename** — Layer1 enriched의 현재 실제 이름은 **`spx_mv_audit_enriched`**(ORM `VAuditEnriched`). 옛 migration의 `spx_v_audit_enriched`/reference doc 표기는 stale. B안 마트 SQL은 `spx_audit_events` 직접 조회라 기능 영향 0.
5. **[검증 데이터 공백 — 블로커] 5/20 시드** — 시드 audit 이벤트 7종에 **`workflow_node_execute`가 없음**. 현재 시드로는 B안 마트가 **빈 결과** → 0-5(시드 재구성) 선행 필수.

## ⚠️ 마이그레이션·배포·머지 함정 (착수 전 필독 — 2026-06-10 소스 확인)

새 Prisma 마이그레이션(마트 UNION 재작성)과 collector 변경을 dev에 머지할 때 아래를 빠뜨리면 **특정 환경에서 조용히 회귀/빌드 깨짐**.

1. **⭐ entrypoint self-heal 하드코딩 리스트 (H-INFRA-01 가족 — 가장 중요)**
   - `dify-audit/scripts/entrypoint.sh` 의 "mart view self-heal" 블록(line 64-73)이 mart 객체 누락 시 **하드코딩된 migration 리스트(line 66)만 재적용**.
   - 현재 리스트: `20260519000000_fix_layer1_mview_naming_and_rbac_prefix` / `20260520000000_add_audit_enriched_tenant_occurred_idx` / `20260520100000_dual_sentinel_actor_dept` / `20260521000000_finalize_audit_enriched_rename`
   - **B안 마트 migration dir 이름을 이 리스트 끝에 반드시 추가**. 안 하면 self-heal이 옛 `spx_mv_model_tokens_daily`(message_send 전용) 정의로 되돌림 → B안 무효화. (`migrate deploy`/baseline 경로는 자동 적용되나 self-heal만 수동)
   - `MART_OBJECTS` 존재 체크(line 52)는 이름 기반이라 동명 재작성은 그대로 통과(추가 불필요).
2. **idempotent 필수 (H-INFRA-01)** — clean boot(`docker compose down -v && up -d`)에서 전 migration 재적용. 마트 재작성은 `DROP MATERIALIZED VIEW IF EXISTS ... CASCADE` + `DO $$ ... END $$`(RBAC 가드) 패턴 — 기존 layer 마이그레이션 그대로 따를 것.
3. **dist 빌드 캐시 (H-INFRA-03)** — collector는 TS→dist 컴파일이 image에 박힘. `restart`로 src 변경 무반영. workflow-nodes.ts 변경 후: **`docker compose build dify-audit && docker compose up -d --force-recreate dify-audit`**. dev 머지 후 동료 안내에도 build 명령 박기.
4. **design↔migration 동반 커밋 (H-DOC-01)** — 이 task/reference 문서 갱신과 migration 파일을 **같은 PR**에. 한쪽만 머지 시 drift.
5. **modify/delete 자동 머지 (5/18 KAN-29 사고)** — 자동 머지가 충돌 마커 없이 dev의 의도적 삭제를 따라가 빌드 조용히 깨짐. **`docker compose build`가 진짜 머지 게이트**(grep/마커만으론 부족). dev 삭제 추적: `git log dev --diff-filter=D --summary -- <경로>`.

> 관련 결함: `hdd/defect-catalog.md` H-INFRA-01/02/03, H-DOC-01, H-MART-01.

## 0단계: 선행 결정/조건

- [x] 0-1. H-DASH-01 좁은 예외 문구 추가 (모델 차원 차트 모델 라벨/호출수는 노드 출처) — `hdd/defect-catalog.md` 갱신 완료 (2026-06-10)
- [x] 0-2. 모델 보유 노드 3종 확정 — `llm` / `question-classifier` / `parameter-extractor` (graphon `enums.py`, 세 노드 모두 `process_data`에 `model_provider`/`model_name`/`usage` 기록)
- [x] 0-3. **마트 통합 vs 분리** — 채택: **기존 `spx_mv_model_tokens_daily` UNION 재작성**(두 모델 차트 한 마트 공유). 이 spec 전체가 이 기준. 대안(전용 mview 신설)은 미채택. → 결정 확정, 구현 완료(PM 최종 확인은 형식)
- [x] 0-4. process_data 오프로딩 대응 — 채택: **1차는 인라인만 파싱**(잘린 노드는 모델 NULL→미분류 잔존 허용, JSON.parse try/catch). `load_full_process_data` 추가 조회는 2차 보류. 오프로딩 빈도 실측은 운영 DB 진입 후(H-DASH-02 영향 크기)

## 1단계: collector 보강 (`dify-audit/src/lib/collectors/workflow-nodes.ts`)

> generated column이 `details->>'modelProvider'`/`modelId`/`totalTokens`를 자동 추출하므로 **추출용 새 마이그레이션 불필요** — 수집기가 같은 키로 채우면 `model_provider_d` 등 자동 채워짐.

> ⚠️ **Prisma collector 주의 (5/14 분석 노트 line 356 + save-event.ts 검증)**:
> - **`prisma generate` 불필요** — collector는 `difyDb.$queryRaw` raw SQL + 수동 result 타입. `n.process_data` 추가 = SQL에 컬럼 + 인라인 타입에 `process_data: string | null`만. Prisma schema/dify-client 재생성 없음.
> - **⚠️ BigInt 직렬화 함정** — `$queryRaw`로 **bigint 컬럼을 직접 읽으면 JS `BigInt` 반환**. details에 담아 `auditDb.auditEvent.createMany`(Prisma JSON 직렬화) 시 **BigInt 직렬화 실패**. → 토큰은 반드시 `process_data.usage.total_tokens`(JSON.parse → JS number)에서 뽑을 것(안전). `n.execution_metadata`나 bigint 컬럼 직접 읽기 금지(읽어야 하면 `Number(x)` 캐스트).
> - **DB 클라이언트 2개 분리** — 읽기 `difyDb`(dify-client) / 쓰기 `auditDb`(audit-client). process_data는 dify DB에서 읽음.

- [x] 1. SQL `SELECT`에 `n.process_data` 추가 (기존 LEFT JOIN 재사용, JOIN 추가 0건)
- [x] 2. node_type 게이팅 — `node_type IN ('llm','question-classifier','parameter-extractor')`일 때만 모델 파싱 (그 외 노드는 모델 키 미주입 → `model_id_d IS NULL`로 자연 제외)
- [x] 3. details 주입:
  - [x] 3-1. `modelProvider: pd.model_provider ?? null`
  - [x] 3-2. `modelId: pd.model_name ?? null`  ← Dify는 `model_name` 키 (주의: 우리 컨벤션은 `modelId`)
  - [x] 3-3. `totalTokens: pd.usage?.total_tokens ?? null`
  - [x] 3-4. JSON.parse 방어 — `process_data` NULL/잘림 시 try/catch, 실패 시 모델 키 미주입
- [x] 4. → **collector 실행 경로 검증(Prisma read→save + BigInt)은 1.5단계 A(V-A1~A5)에서** 수행. (audit 직접 시드로는 collector 경로가 안 타므로 OLTP 시드 + 실제 collector 실행 필요)
- [x] 5. 백필 — 기존 노드 행엔 모델 키 없음. 커서 리셋 재수집 **또는** `details` UPDATE(STORED gen col 재계산). 운영 백필 범위/시점 결정 (방안 결정 완료)
- [x] 5-1. ⭐ **시드 재구성 (검증 블로커)** — `api/scripts/seed/audit_mock.sql`에 `action='workflow_node_execute'` 행 직접 추가(현재 0건). 목업은 audit 직접 시드만 가능(시드 구조 참조). 아래 시나리오 매트릭스로 생성:

  **불변 규칙 (소스 검증 완료 2026-06-10)**:
  - **합 보존**: 챗플로우 `message_send.totalTokens == Σ(그 런의 workflow_node_execute totalTokens)`. 실데이터에서도 성립(iteration/loop 서브엔진 롤업 1회 → 모델노드 각 1회 집계, ref §4-1 + iteration_node.py:222/356). 시드도 정확히 합=노드합으로.
  - **링크 키**: 각 노드 이벤트 + 챗플로우 message_send 이벤트의 `details.workflowRunId`를 **공유**(실스키마 그대로 — 양 collector가 이미 심음). 총합일치 테스트가 message↔노드 페어링에 사용.
  - **공통**: `details.appMode`는 앱별('workflow'|'advanced-chat'), provider 문자열은 기존 message_send 시드와 **동일 포맷**(task 27).

  | 시나리오 | app_mode | 이벤트 구성 | 검증 |
  |---|---|---|---|
  | S1 순수WF 단일 | workflow | node 1 (gpt-4o, 500) | 워크플로우 차트 등장 |
  | S2 순수WF 멀티 | workflow | node 2 (gpt-4o 300 + claude 200), 공유 runId | 1런→모델2개 분해 |
  | S3 챗플로우 페어 | advanced-chat | message_send 1 (model NULL, 800) + node 2 (gpt-4o 500 + claude 300), **합=800**, 공유 runId | 이중집계0(task8)·총합일치(task22)·미분류 해소 |
  | S4 로컬 | workflow | node 1 (ollama/llama3, 400) | "(로컬)" 라벨(task29) |
  | S5 오프로딩 | workflow | node 1 (**model 키 없음**, 100) | '미분류' 잔존 정상 |
  | S6 디버그 | workflow | node 1 (`triggeredFrom='debugging'`) | H-DASH-03 제외 |

  - [x] (a) S1~S6 6종 audit_mock.sql에 추가 (290건, 규모는 차트 안 비칠 정도, 기존 90일 분포 따름)
  - [x] (b) 시드 후 검증: 재시드 → REFRESH → 모델 차트에 워크플로우/챗플로우 모델 등장 + S3 챗플로우 토큰이 노드합과 일치하는지

## 1.5단계: 실행 경로 검증 (정적 시드로 안 잡히는 것)

> ⚠️ **시드 검증 ≠ 실행 경로 검증.** `audit_mock.sql` 시드는 audit에 **직접 INSERT**라 collector(Prisma read→save) / 마이그레이션(entrypoint·self-heal) / 스크립트 실행 경로를 **한 번도 안 탐**. 이 단계가 그 공백을 메움. **"시드 했으니 끝" 착각 금지.**
> 각 항목 **실제 출력(로그/쿼리 결과) 첨부** — 자가 검증 거짓 금지 (delegation-standard §2, 5/15 학습).

### A. collector Prisma 경로 (1단계 직후 실행 가능 — BigInt 함정 포함)

> ▶ **지금(1단계 직후) 실행 순서** — A 전체 + C1 가능. B/C2는 2·2.5단계 후.
> 1. **선행 블로커**: `oltp_mock.sql`에 `workflow_node_executions` 0건 → V-A2 먼저(이게 없으면 collector가 0 events로 끝남)
> 2. V-A2(OLTP 시드) → V-A3(build+force-recreate) → V-A4(로그 BigInt/save 에러 0) → V-A5(결과 행)
> 3. 덤: V-C1(`spx-seed-mock.ps1` 2회 멱등)
> ⚠️ 1단계의 "audit 직접 시드(wn_ 행) 검증"과 **다름** — 1.5-A는 collector가 OLTP 읽어 만든 행 검증(`id NOT LIKE 'wn_%'`로 구분).

- [x] V-A1. **구조 리뷰 (BigInt 거의 결정적)** — `workflow-nodes.ts` SELECT가 **bigint 컬럼을 details에 안 넣는지** 확인. 토큰은 `process_data`(text)→`JSON.parse`→`pd.usage.total_tokens`(JS number)만. `n.total_tokens`/`execution_metadata` 등 bigint 직접 읽기 0건 → BigInt 구조적 불가.
- [x] V-A2. **OLTP 시드** — `oltp_mock.sql`에 `workflow_node_executions` 6건 추가(3종 모델 노드 + code + NULL pd + truncated JSON). ON CONFLICT DO NOTHING 멱등.
- [x] V-A3. **컨테이너 반영** — `docker compose build --no-cache dify-audit && docker compose up -d --force-recreate dify-audit` (H-INFRA-03). ⚠️ **`--no-cache` 필수** (2026-06-10 실측): 일반 `build`는 빌드 캐시가 구 소스로 컴파일 → 변경 미반영. **restart도 금지**(dist 캐시).
- [x] V-A4. **collector 실행 + 로그** — `[workflow_nodes] Collected 6 events`, BigInt/save 에러 0건 (Prisma 저장 경로 실측 통과).
- [x] V-A5. **결과 확인** — Case 1~3(LLM/QC/PE) 모델 키+generated column 정상, Case 4~6(code/NULL/truncated) 모두 NULL(정확 제외).

### B. 마이그레이션 검증 (⚠️ 실행 위치 분리 — 라이브 194에서 파괴적 재시험 금지)

> **V-B1/B2/B3 = 파괴적 리허설**(뷰 강제 DROP/재생성/리셋) → **일회용 DB에서, 194 적용(3.5단계) *전에***. 라이브 공유 194에서 반복 금지.
> **V-B4/9-5 = 읽기전용 확인** → **194 적용 *직후*, 194에서**.
> 실행 순서: [일회용] V-B1·B2·B3 → [194 적용 AP] → [194] V-B4·9-5.

- [x] V-B0. **일회용 audit DB 준비** (V-B1~B3 선행) — 로컬 DB로 리허설(옵션 a). audit 스키마+마트는 194에만 있으므로 택1:
  - (a) 로컬에 일회용 audit DB baseline 적용(제일 깨끗) — `docker/volumes/db/data` 일회용 인스턴스
  - (b) 일회용 없으면 V-B1/B3만 **194 저트래픽 시간에** 짧게(뷰 잠깐 DROP→복구), **V-B2는 194 금지**
  - (c) 리허설 생략 — 멱등 구조+1회 적용 성공+V-B4/9-5만(가장 가벼움, self-heal 미검증 리스크)
- [x] V-B1. **멱등 재실행** [일회용] — 로컬 리허설서 마이그레이션 SQL 2회 실행 에러 0 + REFRESH 후 워크플로우 모델 포함/이중집계 0/미분류 0 확인.
- [ ] V-B2. **clean boot** [⛔로컬만, 194 금지] — `docker compose down -v && up -d` → 전 마이그레이션 재적용, UNION 정의로 생성. ※ 로컬 DB는 바인드 마운트라 `down -v`로 리셋 안 됨 → 진짜 리셋은 `docker/volumes/db/data` 삭제.
- [ ] V-B3. **entrypoint self-heal** [일회용 or 194 저트래픽] — mart 객체 강제 `DROP` 후 dify-audit 재시작 → self-heal이 **새 마이그레이션 포함 리스트**(line 66)로 UNION 정의 복원. (리스트 누락 시 옛 정의로 복원되는지 negative 확인)
- [x] V-B4. **OWNER/GRANT** [194, 적용 직후] — `\dp` 결과 OWNER `audit_writer` + SELECT 권한 정상(AP-7b).

### C. 스크립트 멱등 + drift

- [x] V-C1. **시드 스크립트 멱등** — audit_mock.sql 2회 실행 → wn_ 260/ms_ 30/ae_ 180K 동일, 중복 0.
- [x] V-C2. **drift 게이트** — 194 `\d+` 출력이 B안 UNION 정의(design.md § 2.5.4)와 정합 확인(AP-7b).

> 실행 시점: **A는 1단계 후**, **B는 2단계 후**, **C는 해당 변경 직후**. 세션 분할 시 각 세션 끝에 해당 파트 수행 권장.

## 2단계: 인덱스 + 마트 재작성

> 도구: Prisma Migrate raw SQL (`dify-audit/prisma/audit/migrations/`). `CREATE INDEX CONCURRENTLY`는 트랜잭션 밖 → 수동 실행 (data-mart 5/15 학습).

- [x] 6. 부분 인덱스 추가 — `spx_audit_events_wf_model_idx` ON `(occurred_at, model_id_d)` WHERE `action='workflow_node_execute' AND model_id_d IS NOT NULL`
- [x] 7. `spx_mv_model_tokens_daily` UNION 재작성 (migration `20260610000000_rewrite_model_tokens_daily_union`)
  > **왜 마트 재작성인가(전용 mview/직접쿼리 대비)**: ① 성능 — 일별 사전집계(155행, sub-ms) vs 직접쿼리 시 5/20에 고친 seq scan 1.5s 부활 ② 작업량 최소 — 두 모델 차트가 **여전히 뷰 1개** 조회(전용 mview는 서비스에서 두 소스 병합 추가) ③ 출력 컬럼 동일 → 다운스트림 무영향 ④ `spx_audit_events` 직접 조회로 enriched RBAC 조인 불필요(모델 차트는 부서 차원 없음) → 공유 enriched 뷰 bloat 회피
  - [x] 7-1. (1) 비-챗플로우 메시지: `action='message_send' AND app_mode_d <> 'advanced-chat' AND model_id_d IS NOT NULL`
  - [x] 7-2. (2) 워크플로우+챗플로우 노드: `action='workflow_node_execute' AND app_mode_d IN ('workflow','advanced-chat') AND model_id_d IS NOT NULL`
  - [x] 7-3. `UNION ALL` 후 `COALESCE(model_provider,'미분류')`, `COALESCE(model_id,'미분류')`, `SUM(total_tokens) AS tokens`, `COUNT(*) AS calls`
  - [x] 7-4. 디버깅 제외 — `NOT is_debug` 동등 필터 (`invoke_from_d != 'debugger'` + `triggered_from_d NOT IN ('debugging','rag-pipeline-debugging')`) (H-DASH-03)
    > ⚠️ **drift 감시 (is_debug 중복)**: `spx_audit_events` 직접 조회라 이 필터는 `spx_mv_audit_enriched`의 `is_debug` 정의를 **인라인 복제**한 것. 마이그레이션 주석에 *"Equivalent to enriched.is_debug"* 명시 + 현재 값 동일. **enriched의 is_debug가 바뀌면 이 마이그레이션도 같이 갱신**해야 함(둘이 어긋나면 model_tokens 디버그 필터만 옛 정의). 왜 직접 조회인지(enriched에 `workflow_node_execute` 없음 + 모델 차트는 RBAC/canonical 불필요)는 reference doc §6 참조.
  - [x] 7-5. KST day — `date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul')`
  - [x] 7-6. `UNIQUE INDEX (tenant_id, day, model_provider, model_id)` — CONCURRENTLY refresh 전제. NULL은 SELECT에서 `COALESCE(...,'미분류')`로 사전 치환(표현식 인덱스 불가)
  - [x] 7-7. **`ALTER MATERIALIZED VIEW public.spx_mv_model_tokens_daily OWNER TO audit_writer` + `GRANT SELECT` 보존** (design.md L336 — collector worker만 refresh 권한). 누락 시 refresh chain 실패
  - [x] 7-8. **H-DASH-11 방어 확인** — 쿼리/뷰에서 `process_data` JSON을 **직접 파싱하지 않음**(collector가 1회 파싱 → `total_tokens_d` 등 generated column 경유). 라이브 `::json->>` 금지(30일 범위 수초~수십초 함정)
- [x] 8. ⚠️ **이중집계 차단 확인** — (1)의 `app_mode_d <> 'advanced-chat'`로 챗플로우가 message_send/노드 양쪽에 안 잡히는지 (로컬 리허설서 이중집계 0건 확인)
- [x] 9. 카디널리티 확인 — 모델 보유 노드만 대상이라 출력 행 작음(모델×일×테넌트). REFRESH 스캔 비용은 부분 인덱스로 완화
- [x] 9-1. ⭐ **entrypoint self-heal 리스트 추가 (H-INFRA-01)** — `entrypoint.sh` line 66 리스트 끝에 `20260610000000_rewrite_model_tokens_daily_union` 추가 완료. 마이그레이션 `DROP ... CASCADE` + 멱등.
- [ ] 9-2. → **검증은 1.5단계 B(V-B1~B4)에서** 수행 (멱등·clean boot·self-heal·OWNER)

## 2.5단계: 문서 정합 (H-DOC-01 + drift 게이트 — 마이그레이션과 같은 PR 필수)

> `quality-criteria.md L67` drift 게이트: design.md § 2.5.4 DDL ↔ 실제 마트 DDL **1:1 정합, diff 0**. 불일치 시 작업 중단. 마이그레이션만 머지하고 design 안 고치면 게이트에서 막힘.

- [x] 9-3. **design.md § 2.5.4 갱신** — `spx_mv_model_tokens_daily` DDL을 B안 UNION 정의로 교체 + "message_send만" 문구 정정.
- [x] 9-4. design.md § 2.5 데이터 소스 표 + refresh chain 갱신 — model_tokens가 `spx_audit_events` 직접 의존으로 바뀐 점 반영
- [x] 9-5. drift 게이트 통과 확인 — 194 `\d+` 출력이 design.md DDL과 정합(AP-7b, V-C2)

## 3단계: refresh chain 연동

- [x] 10. `refreshMartChain()`(dify-audit db-poller) 주석으로 의존 변경 명시 (enriched→kpi_calls 체인 + model_tokens 독립). 순차 실행 유지.
- [x] 11. `REFRESH MATERIALIZED VIEW CONCURRENTLY spx_mv_model_tokens_daily` 정상 동작 (194 REFRESH 성공, 12모델 집계)
- [x] 12. dify-audit 컨테이너 재빌드/재시작 후 REFRESH 정상 (워크플로우 노드는 collector 수집 후 유입)

## 3.5단계: DB 적용 (⚠️ 194 공유·라이브 DB — 안전 절차)

> 마트/audit 객체는 **원격 192.168.10.194:15432**(별개 머신)에 있음. 데이터 손실 위험은 없음(파생 뷰 DROP+CREATE + 인덱스, 원천 SELECT만) — **운영상·적용방식**이 핵심.
> 적용 시점: 2·2.5·3단계(마이그레이션+문서+refresh 코드) 완료 후.

### 적용 전 (롤백 대비)

- [x] AP-1. **옛 DDL 롤백용 저장** — `dify-audit/scripts/rollback_model_tokens_old.sql` 생성(zombie enriched 주의 주석 포함). 적용 전 현재 정의를 떠놓기:
  ```sql
  -- 194에서 실행, 결과를 rollback_model_tokens_old.sql로 보관
  \d+ public.spx_mv_model_tokens_daily
  -- 또는 직전 정의(message_send 전용) = 직전 layer2 마이그레이션/ git history 확보
  ```
  깨지면 옛 정의(message_send 전용)를 다시 `CREATE`하면 복구.
- [x] AP-2. **테이블 크기 확인** — 194 `spx_audit_events` = 219MB/173,740행 → 일반 CREATE INDEX로 충분(CONCURRENTLY 선적용 불필요).

### 적용 (migrate deploy = 트랜잭션, 수동 psql 금지)

- [x] AP-3. **`--no-cache` 재빌드** — `docker compose build --no-cache dify-audit` 성공(tsc 0 에러).
- [x] AP-4. **재생성** — `docker compose up -d --force-recreate dify-audit` → entrypoint `prisma migrate deploy`로 194에 적용(트랜잭션·추적). 수동 psql 안 씀.
- [ ] AP-5. **트래픽 낮은 시간대** — DROP→CREATE 윈도우 동안 model_tokens 마트 잠깐 없음 → 그 차트 보는 사용자 잠깐 빈/에러. 공유 DB라 저트래픽 시간 권장.
- [x] AP-6. 로그 확인 — "All migrations have been successfully applied." + self-heal 미실행(mart 객체 존재).

### 적용 전 리허설 (일회용 DB) — 1.5단계 B

- [x] AP-7a. **적용 *전* 로컬 리허설** — V-B1 멱등 2회 + REFRESH 데이터 검증(워크플로우 모델 포함, 이중집계 0, 미분류 0) 통과. (V-B2/B3는 미수행 — 멱등 구조+성공으로 갈음)

### 적용 후 확인 (194, 읽기전용)

- [x] AP-7b. **적용 *직후* 194 검증** — `\dp` OWNER audit_writer / `\d+` UNION 정의 정확 / UNIQUE·부분 인덱스 존재 / REFRESH 12모델. ※ 워크플로우 모델은 collector 수집(또는 3.6 mock 시드) 후 등장 — 194 노드 0건.
- [ ] AP-8. ⛔ **`docker compose down -v`를 194에 쓰지 말 것** — 로컬 전용. 194 적용 검증엔 clean-boot 불필요.

### 롤백 (만일)

- [ ] AP-9. 차트/refresh 깨지면 → AP-1의 옛 DDL로 `spx_mv_model_tokens_daily` 재생성 + `prisma migrate resolve` 정합. 원천 데이터는 무관(영향 없음).

## 3.6단계: 194 mock 시드 적용 (대시보드 가시화)

> 대시보드는 **194를 읽는데** 194엔 `workflow_node_execute`가 (collector 미수집이라) 0건 → 차트에 워크플로우/챗플로우 모델 안 뜸. **194에 mock 직접 INSERT는 허용**(이미 mock 환경, 행 추가일 뿐) → 로컬과 동일하게 `audit_mock.sql` S1~S6를 194에 넣어 가시화.

### A. 현 상태 확인 (시드 전)

- [x] MS-1. **194 audit_events 현황** — `workflow_node_execute` 0건, `wn_` 0건, `ms_` 0건(시드 미적용). `ae_` 140,360건 + 실수집 33,438건 = 총 173,798건. `message_send` 84,214건.
- [x] MS-2. **마트 현황** — 12개 모델(message_send 기반만). 워크플로우/챗플로우 모델 미등장 확인(baseline).
- [x] MS-3. **audit_mock.sql과 194 차이** — 194에 `wn_`/`ms_` 0건 → S1~S6 전량 INSERT 필요. `ae_` 140K는 이미 존재.

### B. 시드 방법

- [x] MS-4. **범위 결정** — **S1~S6(`wn_`/`ms_`) 부분만** 194에 적용 (ae_ 180K 안 건드림). 가시화 목적 충분.
- [ ] MS-5. **적용** — 선택 범위의 INSERT를 194에 실행. 맨 위 `DELETE ... id LIKE 'wn_%'/'ms_%'`로 멱등(재실행 안전). generated column 자동 채워짐.
- [ ] MS-6. **REFRESH** — `REFRESH MATERIALIZED VIEW CONCURRENTLY public.spx_mv_model_tokens_daily;` (UNIQUE INDEX 전제. db-poller 5분 주기 자동 REFRESH도 있으나 즉시 보려면 수동).

### C. 영향도 확인

- [ ] MS-7. **DELETE 스코프 = 실데이터 무영향** — DELETE는 `wn_`/`ms_`(+전체 적용 시 `ae_`) **접두사만** → 194 실데이터/중요 객체 안 건드림. `DROP`/`TRUNCATE` 없음. 확인: 시드 전후 비-mock 행수 동일.
- [ ] MS-8. **차트 변화** — REFRESH 후 MS-2 재실행 → 워크플로우/챗플로우 모델 등장(llama3 "(로컬)", 멀티모델 분해, 챗플로우 재분류). 대시보드 모델 차트 + model-call-share 반영.
- [ ] MS-9. **collector 실데이터와 공존** — 나중에 collector가 실제 `workflow_node_execute` 수집해도 `wn_` mock과 접두사로 구분되어 공존(중복/충돌 없음). 단 mock+실데이터 혼재 시 수치 합산됨을 인지.
- [ ] MS-10. **전체 적용 선택 시** — audit_mock.sql 전체면 `ae_` 180K 베이스도 재시드 → 194 기존 mock 통계 전반 영향. S1~S6만 적용이면 모델 차트만 영향(권장).

## 3.7단계: ⚠️ 194 DB 손상 정리 + self-heal 경화 (H-INFRA-01 재발 — B안 중 발견)

> **발견(2026-06-10)**: 194에 `spx_mv_audit_enriched` **중복(2개)** + zombie `spx_v_audit_enriched` + `spx_mv_model_tokens_daily` **누락**. = 마이그레이션 부채(H-INFRA-01)가 entrypoint self-heal 부분 실행으로 표면화.
> **근본 원인**: model_tokens 누락 → self-heal이 하드코딩 리스트를 `psql -f`로 재적용 → 20260519의 `DROP ... CASCADE`가 Layer2 먼저 제거 → 뒤 단계 에러 시 `ON_ERROR_STOP=1`+`set -e`로 HALT → model_tokens 재생성 전 멈춤 + enriched 중복/zombie 누적. 재시작마다 반복 위험.
> **B안 무관 부분**: enriched 중복/zombie는 B안 작업 부작용 아님(기존 부채). 단 self-heal이 model_tokens를 또 날릴 수 있어 **재발 방지 필요**.

### 임시 복구 (완료)

- [x] DR-1. **model_tokens 복구** — 20260610 마이그레이션 SQL 재실행. B안 디커플링(`spx_audit_events` 직접 의존) 덕에 enriched 엉망과 무관하게 복구됨(멱등). `_prisma_migrations` 상태 영향 없음.

### 진단 (정리 전 필수) — ⛔ 194 복구 후 (현재 DB 다운, 빨라야 내일)

- [ ] DR-2. [194 대기] **enriched 현황** — `SELECT schemaname, matviewname FROM pg_matviews WHERE matviewname LIKE '%audit_enriched%';` → 중복이 동일 스키마인지/다른 스키마(audit 잔재 등)인지, zombie `spx_v_audit_enriched` 위치 확인.
- [ ] DR-3. [194 대기] **live 식별** — `kpi_calls`/drill 서비스·뷰가 **어느 enriched에 의존**하는지(`\d+ spx_mv_kpi_calls_daily` 정의의 FROM). 그게 live, 나머지는 제거 대상.

### 정리 (⚠️ CASCADE 주의) — ⛔ 194 복구 후

> ⚠️ XX001 스토리지 손상이 진짜 원인일 수 있으므로(객체 드리프트가 그 증상일 가능성), 194 복구 시 **인프라가 디스크/백업 확인 후** 어느 relation이 깨졌는지부터 파악. 마트면 재생성, 실데이터/카탈로그면 백업 복원.

- [ ] DR-4. [194 대기] **zombie + 중복 제거** — live 아닌 `spx_v_audit_enriched`(zombie) + 중복 `spx_mv_audit_enriched`만 `DROP MATERIALIZED VIEW ... CASCADE`. ⚠️ **live를 잘못 드롭하면 kpi_calls까지 CASCADE로 날아감** → DR-3 의존성 확인 후.
- [ ] DR-5. [194 대기] **정리 후 검증** — `pg_matviews`에 enriched 1개(`spx_mv_audit_enriched`)만 / zombie 0 / kpi_calls·model_tokens 정상 REFRESH / 대시보드 차트 정상.

### 재발 방지 (self-heal 경화) — 로컬 완료

- [x] DR-6. **self-heal 부분 실패 안전화** — `entrypoint.sh` self-heal 블록에서 `ON_ERROR_STOP=1` 제거 + `if ! psql ...; then echo WARNING; fi`로 변경 → 개별 마이그레이션 에러가 후속(특히 20260610 model_tokens)을 막지 않음. ⚠️ 트레이드오프: 에러가 WARNING으로만 남으므로 **self-heal 후 마트 수동 검증 필요**.
- [x] DR-7. **enriched 마이그레이션 멱등성 점검** — 로컬 DB에서 self-heal 리스트 2회 연속 실행 → enriched 1개/zombie 0/Layer2 정상. 로컬 멱등 확인.
- [x] DR-8. **defect-catalog H-INFRA-01b 신설** — "self-heal 부분 실행이 CASCADE로 mart 추가 손상" 사례·방어 추가.

## 4단계: 백엔드 서비스 repoint (2종)

- [x] 13. ORM — `mart.py` `MvModelTokensDaily`. `tokens`·`calls` 존재 + ⚠️ **타입 버그 수정**(로컬 e2e서 발견): `tenant_id` `StringUUID`→`sa.Text`(마트 UNION 출력은 text), `tokens` `BigInteger`→`sa.Numeric`(SUM 결과). docstring UNION으로 갱신.
- [x] 14. `dashboard_model_tokens_service.py` — `MvModelTokensDaily` 그대로(코드 변경 0, 자동 유입). docstring "message_send only"→"UNION" 갱신.
- [x] 15. `dashboard_drill_calls_service.get_model_call_share` **repoint** 완료
  - [x] 15-1. `VAuditEnriched` 직조회 **완전 제거** → `MvModelTokensDaily` `SUM(calls)` GROUP BY provider/model_id + `_LOCAL_PROVIDERS` "(로컬)" 라벨(H-DASH-09)
  - [x] 15-2. share_percent 계산 유지, 모집단 마트로 통일
  - [x] 15-3. `get_dept_call_count`/`_rps`(부서 차원) 미터치 — kpi_calls 유지

## 5단계: 프론트엔드

- [x] 16. 모델별 토큰 차트 — 하드코딩 "미분류 (Workflow)" 버킷/`COLOR_UNCLASSIFIED`/`unclassified_workflow_tokens` 제거 **이미 완료**(6/10 작업3, 충돌#2). 잔존 주석 1줄(`index.tsx:21`)만 정리. → 실제 모델 표시 확인만 남음. 로컬 "(로컬)" 라벨 유지(H-DASH-09)
- [x] 17. 모델별 호출 점유 차트 — 마트 기반으로 워크플로우/챗플로우 모델 자동 등장(API display_name 그대로, 프론트 코드 변경 불필요)
- [x] 18. ⚠️ **단위 툴팁 추가** — `model-call-share.tsx` 제목 `title` 속성: "모델 호출 수 = 노드 단위, 워크플로우 1회가 여러 모델 호출로 집계되어 KPI 총 호출량과 다를 수 있음"
- [x] 19. 로딩 상태 골격 유지 패턴 수정 — early-return 제거 → 카드+제목 유지 + `h-[250px] animate-pulse` 스켈레톤
- [x] 19-1. **기존 차트 컨벤션 준수** — 골격 유지(5/19 패턴 B 버그 방지)/고정 높이/숫자 포맷/단위 툴팁 정합. 스키마 무변경이라 대부분 자동.

## 5.5단계: 로컬 E2E 검증 (4·5 코드 완료 후 — 194 다운 중 가능)

> **순서**: 4·5단계 코드·단위테스트 완료 → **여기(로컬 e2e)** → (194 복구 후) 라이브 정합.
> 로컬 DB엔 AP-7a 리허설 때 **UNION 마트 + audit_mock S1~S6(`wn_`) 시드**가 있음(V-B1/V-C1) → 워크플로우 모델이 로컬 마트에 존재. **api를 로컬 DB로 임시 전환**하면 4·5를 화면까지 검증 가능. **검증 후 원복 필수.**
> ⚠️ DB 호스트를 바꾸면 **대시보드 전체가 로컬 mock을 읽음**(모델 차트뿐 아님) → 로컬에 RBAC/audit 전체 시드(spx-seed-mock.ps1)가 있어야 다른 차트도 안 깨짐.

### 셋업 (임시)

- [x] LV-1. **api DB 접속 위치 확인 + 현재값 기록** — `docker/.env` `DB_HOST=192.168.10.194`/`DB_PORT=15432` 기록(원복용).
- [x] LV-2. **로컬 전환** — DB 호스트 → `db_postgres:5432` + api recreate.
- [x] LV-3. **전제 확인** — 로컬에 UNION 마트(ollama/llama3 포함) + RBAC 17부서 mock 확인.

### 검증

- [x] LV-4. **4단계 서비스** (flask app_context 직접 실행) — model_tokens 13개 모델 `llama3 (로컬)` 포함 총 250M / model_call_share 13개 모델 "(로컬)" 라벨·share_percent 정상. ✅
- [x] LV-5. **5단계 프론트** — 코드 리뷰로 골격/고정높이/숫자포맷/툴팁 정합 확인. (시각 확인은 web 빌드+브라우저 필요 — 미수행)

### 원복 (필수)

- [x] LV-6. **DB 호스트 원복** — 194로 복원 + api recreate.
- [x] LV-7. **원복 확인** — `.env` diff 0 (로컬 전환 흔적 없음). ※ 194 정상 연결은 DB 복구 후 확인.

## 6단계: 테스트 + grep 게이트

- [x] 20. H-DASH-01: 챗플로우 토큰 노드 1회만(이중집계 0) — 단위테스트 assertion 커버
- [x] 21. H-DASH-02: WORKFLOW/챗플로우 모델 실제 등장(미분류 아님) — `test_workflow_models_included`(양 서비스)
- [x] 22. 차트 간 총합 일치 — 로컬 S3 챗플로우 `node_tokens 24,000 == msg_tokens 24,000`, 미분류 0건 ✅
- [x] 23. 단위 차이 — 호출 점유 단위 테스트 커버
- [x] 24. H-DASH-03: 디버그 마트 레벨 제외 — `test_*_debug` 신설
- [x] 26. ⚠️ **테스트 재작성 완료** — `test_dashboard_model_tokens_service` 전면 재작성(H-DASH-01/02/03/09), `test_dashboard_drill_calls_service::TestModelCallShare` 마트 기반 목(3컬럼 `(provider, model_id, count)`)으로 교체. **pytest 15 passed**.
- [ ] 27. [⛔ 194 대기 — 실데이터 필요] **provider 포맷 일치 spot-check** — 노드 `process_data.model_provider`(= `model_instance.provider` = `configuration.provider.provider`)와 `messages.model_provider`가 **같은 모델에 같은 문자열**인지(플러그인 네임스페이스 `langgenius/...` 여부). 불일치 시 한 모델이 막대 2개로 쪼개짐 → 정규화 필요. ※ 로컬 시드는 우리가 맞춘 합성이라 무의미 → 194 collector 실수집 데이터로 대조.
- [ ] 28. [⛔ 194 대기 — 실데이터 필요] **중첩 노드 이중집계 검증** — iteration/loop/tool 노드도 usage 누적하나 `model_provider` 미기록 → `model_id_d IS NOT NULL` 필터로 자연 제외(이론). ※ 로컬 시드(S1~S6)에 중첩 구조 없음 → 실제 중첩 워크플로우(194)로 inner LLM 노드 합 == 실행 총 토큰 일치 실측.
- [x] 29. **로컬 모델 판별** — `test_local_provider_label`: bare `ollama` → "(로컬)" 통과. ⚠️ **단 운영(159)은 `langgenius/ollama/ollama` namespaced라 실제론 라벨 안 붙음(버그)** → 테스트가 bare라 못 잡음. **정규화 fix 필요**(PROMPT.md 로컬판별 조사 / 아래 메모).
- [x] 25. grep 게이트 (3/3 통과)
  - [x] 25-1. `VAuditEnriched|enriched` in drill_calls_service → **0 hit** (enriched 직조회 완전 제거)
  - [x] 25-2. `FROM (messages|workflow_runs|workflow_node_executions)` in services/admin/ → **0 hit**
  - [x] 25-3. `process_data` in workflow-nodes.ts → **7 hit** (수집기 보강 확인)

> ⚠️ **테스트 맹점 (2026-06-10 159 실데이터 대조)**: 26·29의 로컬 라벨 테스트가 **bare `ollama`로 통과** → green이지만 **운영 namespaced(`langgenius/ollama/ollama`, `yangyaofei/vllm/vllm`)에선 라벨 0건**. 정규화(마지막 세그먼트) + `vllm` 추가 fix 후, **테스트 입력도 namespaced로 바꿔야** 회귀 방지됨. (PROMPT.md 로컬판별 조사 + references/local-provider-classification.md)

## 검증 체크리스트 (단계별 게이트)

| 단계 | 통과 조건 |
|------|---------|
| 1단계 collector | workflow_node_execute 행의 model_*_d가 3종 노드에 채워짐 |
| 2단계 마트 | 챗플로우 토큰이 message/노드 한쪽으로만 (이중집계 0) |
| 3단계 refresh | CONCURRENTLY refresh 성공 + 신규 행 |
| 4단계 service | model-call-share가 마트 조회, enriched 직조회 0 |
| 5단계 프론트 | 워크플로우/챗플로우 모델 등장 + 단위 툴팁 |
| 6단계 테스트 | model_tokens 합 == kpi_calls 챗플로우 합, 호출 점유 합 ≥ 총 호출량 |

## 참조

- 근거 조사: [[3. 프로젝트/spx-agent/references/workflow-model-classification.md]]
- 결함: [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] H-DASH-01 / H-DASH-02
- 마트 인프라(상위): [[3. 프로젝트/spx-agent/hdd/specs/tasks/data-mart.md]]
- 1차 접근(보존): [[3. 프로젝트/spx-agent/hdd/specs/tasks/model-tokens.md]]
- audit 필드 명세: [[3. 프로젝트/spx-agent/references/audit-details-spec.md]]
