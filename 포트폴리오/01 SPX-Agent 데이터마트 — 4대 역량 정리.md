---
tags:
  - 포트폴리오
  - 커리어
  - 데이터파이프라인
  - spx-agent
created: 2026-09-20
updated: 2026-09-20
---

# SPX-Agent 데이터마트 — 4대 역량 정리

> [!info] 위치
> [[포트폴리오/00 성과 뱅크.md]] § "1. 데이터마트 아키텍처"의 **상세 근거 노트**.
> 성과 뱅크는 3~5줄 요약, 이 노트는 면접·서류에서 꺼낼 수치와 서사를 담는다.

> [!warning] 정직 기준 (성과 뱅크 원칙 준수)
> 코드 생성은 Claude Code가 했다. 이 노트에 적는 건 **내가 판단·검증·표준화한 부분**이다.
> - ✅ **내 것으로 말해도 되는 것**: 설계 대안 비교와 채택 근거, SQL 실측·검증, 스코프 판단(무엇을 없앨지), 표준 정의
> - ⚠️ **"AI로 구현"이라 말해야 하는 것**: DDL·서비스 코드·프론트 컴포넌트의 타이핑 자체
> - 판별 기준: **왜 이렇게 설계했는지 내 언어로 설명 가능한가** → 아래 4개 항목은 전부 가능

**기간**: 2026-05-04 ~ 2026-06-15 (데일리 노트 24건 · 마이그레이션 13개 · 회의 결정 3회)

---

## 한 줄 요약 (서류용)

> Dify 셀프호스트 운영 대시보드의 데이터 마트를 설계·표준화. 설계안을 3차례 전면 재검토한 끝에 **신규 테이블 없이 2계층 4객체 구조**로 확정했고, EXPLAIN ANALYZE 실측으로 병목을 3단계 좁혀 **p95 3,333ms → 420ms(최대 830배)** 개선. 위젯 후보 10개·데이터소스 12종·차트 드로어 4종을 근거를 대고 잘라내 범위를 확정.

---

## 1. 마트 구조 설계·표준화

### 설계안 3차 전환 — 최종적으로 신규 테이블 0개

| 시점 | 안 | 결과 |
|---|---|---|
| 5/8 | 4-fact (신규 테이블 4개 + 별도 ETL) | 폐기 |
| 5/11 | OLTP 3-fact | 폐기 |
| 5/12 | **audit 단일 SoT + RBAC JOIN** | ✅ 확정 — **신규 테이블 0개** |

```
[원본]    public.spx_audit_events   (Generated Column 8개 보강)
           + RBAC 5종 (spx_departments / spx_department_members / spx_resource_ownership …)
              │
[Layer 1]  spx_mv_audit_enriched             MView · 18컬럼 · 이벤트 1건 = 1행
           spx_v_resource_ownership_enriched View  ·  7컬럼 · state 스냅샷
              │
[Layer 2]  spx_mv_kpi_calls_daily            MView · 11컬럼 (차원 6 + 측정 5)
           spx_mv_model_tokens_daily         MView ·  6컬럼 (차원 4 + 측정 2)
```

**MView / View 선택 기준을 규모로 명시** — enriched는 900만 행 + 5분 stale 허용 + 5개 컴포넌트 공용 → MView. ownership은 수천 행 + 부서명 즉시 반영 필요 → View.

### 정량

| 항목 | 수치 |
|---|---|
| 마트 객체 | **4종 / 2계층** |
| Generated Column | **8개** (`_d` 접미사 7 + `target_app_id` 예외 1) |
| 인덱스 | 원본 **13개**(기존 8 + 마트 가속 5) + Layer 1·2 **5개** |
| Prisma 마이그레이션 | **13개** 중 마트 전용 **11개** |
| 모니터링 메트릭 | **6개** (collector lag / mart lag / REFRESH 소요 / 행수 차이 / dept NULL / actor NULL 5%) |
| 백엔드 전환 | 신규 `api/models/mart.py` **152줄**, 수정 **10파일**, `query_helpers.py` 헬퍼 **7개 전부 제거** |
| 쿼리 단순화 | 소스 테이블 **8종 + 최대 9-CTE** → 마트 **4객체 SELECT** |
| SLA | 마트 신선도 **< 7분** · collector lag **< 10분** · 첫 페인트 **< 3초** |

### 내가 정의한 표준

- **네이밍** — `audit`/`mart` 별도 스키마 폐기 → `public` 단일 + **`spx_` 접두사**. MView는 `spx_mv_`, View는 `spx_v_`. 잘못 붙은 `spx_v_audit_enriched` → `spx_mv_`로 바꾸는 **전용 마이그레이션까지 작성**
- **공통 컬럼** — details JSON 추출 컬럼 전부 `_d` 접미사. 출처 자체기록 + 미래 top-level 컬럼과 충돌 방지
- **날짜** — `date_trunc('day', occurred_at AT TIME ZONE 'Asia/Seoul')` 단일 경계 → 모든 기간 비교가 `SUM` 한 번
- **부서 차원** — 옵션 A~D 비교 후 **B(ID만 사전 JOIN, 부서명 query-time)** 채택 → 부서명 변경 즉시 반영. `actor_dept_id`(채택 폭) / `app_owner_dept_id`(호출량) **둘 다 GROUP BY 보존**
- **sentinel UUID 2종** — `…000000000000`=미배정 / `…ffffffffffff`=외부.
  근거가 기술적으로 명확: PostgreSQL은 표현식 인덱스에 `REFRESH CONCURRENTLY` 불가 → SELECT 단계에서 NULL을 치환해 인덱스를 단순 컬럼으로 유지
- **타입 가드 (H-MART-01)** — `actor_type IN ('account','end_user')`일 때만 `::uuid` 캐스트.
  mock에 account만 있어 테스트는 통과하지만 운영의 `'system'` / `'chatbot-public-001'`에서 터질 결함을 **운영 진입 전 차단**
- **비즈니스 규칙을 마트가 흡수** — "컴포넌트는 SELECT만 한다"가 황금 규칙. `is_debug`(H-DASH-03) · `is_canonical_call`(H-DASH-01)을 플래그 컬럼으로 박음

### 규약을 코드로 강제한 부분 (면접 포인트)

`architecture.md`에 **마트 인터페이스 의무화(5/18)** 명문화 — raw 테이블 직접 SELECT ❌ / `details->>` raw 추출 ❌ / `_d` 컬럼 직접 참조 ❌.
→ **grep 게이트 4종**을 PR 조건으로 걸어 위반 3종 **0 hit**, 정상 참조 **22 hits** 통과.
→ "미배정" 하드코딩도 SQL 5곳 + Python 2곳 → 중앙 상수로 통합해 **grep 0건**.

---

## 2. SQL로 데이터 직접 탐색·검증

### 성능 — EXPLAIN ANALYZE 기반, 최대 830배

엔드포인트 **15종 전수 EXPLAIN ANALYZE 매트릭스**를 만들어 병목을 특정.

| 개선 | Before | After | 배수 |
|---|---|---|---|
| p95 응답 (인덱스 추가) | 3,333ms | **420ms** | 87%↓, Seq Scan **7 → 0** |
| model-call-share (시드 재구성) | 1,480ms | **57ms** | **26배** |
| dept-user-activity (CTE 통합) | 5,800ms | **7ms** | **830배** |
| 서버 DB 재측정 (7일 윈도우) | 5,800ms | **12.1ms** | **480배** |

**진단이 다층적이었던 게 핵심** — 이 서사가 면접에서 제일 잘 먹힘:
1. 5/19 약식 측정 → "N+1 7쿼리가 원인"
2. 5/20 단일 CTE 통합 후에도 **5.9초** → N+1은 0.1초 영향뿐
3. 진짜 원인 = base CTE **437K행 정렬 시 work_mem 4MB 초과 → `external merge Disk: 4496kB`**
4. 더 나아가 DB 0.76~420ms vs HTTP 1.45~5.8s(**10~100배 차이**) 대조 → **"DB가 아니라 Gunicorn 단일 워커가 병목"** 결론
5. → 추가 인덱스(INCLUDE) 제안을 **오히려 기각**

### SQL이 아니었으면 못 잡았을 것들

| 발견 | 내용 |
|---|---|
| RBAC 테이블 존재 오인 | "타 담당자 ETA 대기"로 이틀 낭비 → `\dt \| grep department`로 이미 DB에 5종 잔존 발견. 이후 **H-DASH-17(관찰 부족 함정)** 정식 등록 + "운영 인프라 검증" 절차 신설 |
| 테이블 접두사 2회 정정 | `sp_`(가정) → 무접두(가정) → **`spx_`(확정)**. 라이브 DB `\dt` 직접 대조로 종결 |
| '미분류' 토큰의 정체 | 워크플로우가 아니라 **챗플로우(advanced-chat)** — message의 모델 컬럼이 NULL로 생성됨. 순수 워크플로우는 아예 누락 |
| `LOCAL_PROVIDERS` 운영 라벨 0건 | 운영은 `langgenius/ollama/ollama` 형식인데 코드는 bare 매칭 → **시드·테스트가 bare라 green이지만 운영에선 라벨 0건** |
| vLLM 전제 오류 | 전제("openai_api_compatible로 등록")와 달리 독립 플러그인. `provider_type`도 `custom`/`system`(자격증명 범위)이라 **로컬 판별 신호로 사용 불가** |
| 디버깅 대상 DB 오인 | api 컨테이너는 원격 DB에 연결. 로컬 DB를 보며 헤맴 → `docker exec … env \| grep DB_` 필수 절차화 |
| ORM ↔ 실제 타입 불일치 2건 | `tenant_id` StringUUID→Text, `tokens` BigInteger→Numeric |

### 정합성 검증 수치

- **계층 합 대조**: Layer 1 **292** = Layer 2 calls 합 **292** 완전 일치 / Layer 1 316건 **100% 매칭**
- **시드 카디널리티**: 450,156 / 6,522 / 155 — 핵심 마트 3종 정합 통과
- **idempotency**: 시드 2회 실행 → `177,638 → 180,000` 자동 정합, **중복 0**
- **실측으로 컬럼 선택**: `created_by` 매핑률 **29%** vs `owner_account_id` **94.7%(18/19)** → top-owners를 후자 기준으로 확정
- **운영 모델 타입 분포**: `text-generation 9` / `embeddings 4`, rerank·STT·TTS·moderation **등록 0건**
- **게이트**: collector 6케이스 전수 + **pytest 15 passed / 0 failed** + **grep 3/3**

---

## 3. 원천 ~ 마트 전체 흐름 설명

### 8단계 파이프라인

```
[1] 원천   Dify OLTP (messages / workflow_runs / workflow_node_executions / apps / conversations …)
           RBAC 5종 · Keycloak 4테이블 (user_entity / groups / user_group_membership / credential)
              │  수집 경로 14종 = DB폴링 collector 13 + nginx log-watcher + pg_trigger 5종 + self-audit
[2] 적재   public.spx_audit_events        (source 5종으로 수집 경로 구분)
[3] 보강   Generated Column 8개 + 마트 가속 인덱스 5개
[4] L1     spx_mv_audit_enriched          ← ownership INNER JOIN + members LEFT JOIN
[5] L2     spx_mv_kpi_calls_daily / spx_mv_model_tokens_daily
[6] 백엔드 api/models/mart.py → services/admin/dashboard_*_service.py (9파일 1,897줄)
[7] API    GET /console/api/dashboard/{kpi, dept-objects, model-tokens, dept-activity, drill/{metric}/{chart}}
[8] 프론트 use-admin-dashboard.ts / use-admin-drill.ts (TanStack, staleTime 5분)
           → /dashboard/page.tsx → KPI 4 + 차트 2 + 표 1 + drill 차트 8 + drill 표 4 (ECharts)
```

| 구간 | 수치 |
|---|---|
| 수집 경로 | **14종** (collector 소스 16개 중 유틸 2 제외) |
| DB 폴링 collector | **13종**, 5분 주기, 상태는 `spx_collector_state` 기록 |
| pg_trigger | **5종** (documents / datasets / apps / api_tokens / tenant_account_joins) |
| 원천 테이블 | Dify OLTP **5** + RBAC **5** + Keycloak **4** |
| API 라우트 | **17개** (KPI 4 + drill 12 + drawer 1) |
| 프론트 | KPI 4 + 메인 차트 2 + 표 1 + **drill 차트 8 + drill 표 4** |

### 동기화 — 구간별 지연을 수치로 관리

| 구간 | 방식 | 주기 |
|---|---|---|
| Dify OLTP → audit | collector 폴링 | 5분 |
| Dify DELETE/UPDATE → audit | `spx_log_dify_change()` pg_trigger | 실시간 |
| nginx access.log → audit | log-watcher (offset 추적) | 실시간 |
| audit → 마트 | `refreshMartChain()` CONCURRENTLY ×3 순차 | 5분 (collector 사이클 끝) |
| 마트 → 프론트 | TanStack staleTime | 5분 |

**pg_cron을 안 쓴 이유를 흐름으로 설명할 수 있음** — mat별 분리 cron이면 collector 5 + enriched 5 + mat 5 = **최악 15분**. collector 사이클 끝에 chain을 붙이면 **mat 단계 추가 지연 0분 → 최악 10분**. 그리고 "전체 lag = MAX(병목 단계)이고 진짜 병목은 collector 5분 폴링이라 마트만 실시간으로 만들어도 체감 0"까지 문서화. 구현 **+16줄**, try/catch 격리로 마트 실패가 collector 다음 회차를 막지 않게 처리.

**도구·권한 분리도 흐름의 일부** — DDL은 Prisma migration 소유(Alembic 아님), `mart.py`는 `info: {'is_view': True}` 마커로 Alembic autogen 제외. `audit_writer`는 ALTER 권한이 없어 superuser DDL + `prisma migrate resolve --applied` 패턴.

**흐름 재설계까지 수행 (B안, 6/10)** — 워크플로/챗플로 모델이 차트에 안 잡히는 문제를 `spx_mv_audit_enriched` 의존 → **`spx_audit_events` 직접 조회 + UNION ALL 2갈래**로 재작성. 모델 차트는 부서 차원이 없어 RBAC JOIN이 불필요하다는 판단으로 refresh chain에서 독립. 챗플로우 이중집계는 (1)에서 제외하고 (2) 노드에서만 집계 → 검증 `node_tokens(24,000) == msg_tokens(24,000)`, 미분류 **0건**.

---

## 4. "무엇을 만들고 없앨지" 설계

### 후보 대부분을 근거를 대고 잘라냄

| 대상 | 후보 | 채택 | 판단 |
|---|---|---|---|
| 추가 위젯 | **10개** | **0개** | 확정 범위 미진입 |
| 드릴 차트 데이터 소스 | **12종 비교** | **전부 폐기** | audit 단일 SoT로 통일 |
| 차트 드로어 | **4종** | **0종** | all-or-nothing 기준 선설정 후 제거 |
| context-bar 구성요소 | **6개** | **0개** | 5개 제외 + 1개 보류 → 컴포넌트 자체 보류 |
| model_type | **6종** | **2종** | 4종 구조적 불가 + 실익 0 |
| 외부 연결 제거 | docs 7종 | **10종 전부** | 코드에서 3종 추가 발견 |
| docs 페이지 | ~102p | **23p(22%) 삭제** | 애매한 "검토" 버킷 27 → 0 |

### 판단 근거가 좋은 사례 (면접용)

**차트 드로어 4종 전체 제거** — "4개 다 의미 있어야 살아남는다, 하나라도 아니면 전체 제거"라는 **판정 기준을 먼저 세우고** 검증. KPI 4번 드로어 지표(호출/사용자/에러율/마지막 사용)가 **하단 표 컬럼과 100% 일치**하는 걸 발견해 격하. 좌하 차트 클릭 트리거까지 함께 제거.

**context-bar 5개 요소 제외 사유가 전부 "중복 회피"** — 기간 칩은 `DashboardControls`에 이미 있음 / URL 북마크는 query string으로 이미 deep link / 새로고침 이미 있음 / 활성 부서 칩은 차트·표에 명시. 남은 책임도 **KPI 카드 active ring + ESC 토글로 자연 분산 흡수** → 렌더링 자체가 잉여.

**모델 토큰 커버리지 — 구조적 불가를 "명시적 제외로 문서화해 재론 방지"**
- 6종 중 토큰(`usage.total_tokens`) 개념이 있는 건 **LLM·임베딩 2종뿐**. 나머지 4종은 invoke 반환 엔티티에 usage가 정의조차 안 됨
- 운영 DB 실측으로 **등록 0건** 확인 → 구조적 불가 + 실익 0 **이중 확인**
- Phase 1(임베딩 인덱싱 편입) 권고 — 고가치(유료 embedding, **44.5만 토큰** 실재) · 저비용(collector **+6줄**)
- Phase 2(쿼리타임) 보류 — **496건** 실재하나 코어 수정 필요 → 무수정 원칙 위배

**외부 연결 제거 — 두 방식의 규모를 산정해 결정**

| 방식 | 규모 | 위험 | 원칙 |
|---|---|---|---|
| 직접 삭제 | **24파일 / ~2,000줄** + DB 마이그레이션 동반 | 7/10 | ❌ 원본 무수정 위반 |
| **피처플래그** | **env 4개 + 원본 ~20줄** | 3/10 | ✅ 가역적 |

항목별 차단 가능성까지 %로 산정(Website 90% / API Extension 85% / 외부 KB 70% / 트레이싱 30%). Twitter는 전수 grep **0건** → **작업 불필요로 판정**.

### 실제 삭제 실적

- **dead code 8건 일괄 제거** — 프론트 훅 3 + 타입, 백엔드 메서드 2 + 응답 타입 4 + 컨트롤러 2, 테스트 1 전면 재작성
- **model-tokens 죽은 코드** — 발견 경위가 좋음: **API가 안 보내는 필드를 프론트 타입이 들고 있어 `undefined===0`이 false → 빈 상태 조건이 영영 안 걸림**
- **lint 부채 70건 → 9건** (자동 54 + 수동 7)
- **base docker-compose 38줄 제거**

### 없애지 **않기로** 한 판단도 기록 (균형 근거)

- `controllers/console/admin/` — 전제("빈 폴더")가 오류. 실제로는 원본 관리자 모듈(17KB) → **삭제 스킵**
- `ja/`·`zh/` — 삭제 대신 **nav 비활성**. "회복 비용 비대칭 — 삭제는 orphan이라 git 복원 불가, nav 비활성은 가역"
- fallback 테스트 3건 — 삭제했다가 **복원**. "서비스에 try-except가 없어서" 가 아니라 **H-DASH-04 graceful degrade를 잃은 채 방치된 걸 테스트만 지워 덮은 것**으로 재판단 → 4파일 11메서드 + 테스트 11건 복원

---

## 면접 예상 질문 → 답변 뼈대

| 질문 | 답변 축 |
|---|---|
| 마트를 왜 MView로 했나 | 행수·stale 허용·공유 컴포넌트 수 3축으로 판단. enriched 900만 행·5분 stale OK·5개 컴포넌트 공용 → MView / ownership 수천 행·즉시 반영 → View |
| 왜 신규 테이블을 안 만들었나 | 4-fact·OLTP 3-fact를 검토했으나 별도 ETL 유지비 대비 이득이 없었음. audit이 이미 단일 SoT라 RBAC JOIN만으로 충분 |
| 성능 개선의 진짜 원인은 | N+1 → work_mem 부족(디스크 정렬) → Gunicorn 단일 워커. 3단계로 좁혔고 마지막엔 DB 인덱스 추가 제안을 기각 |
| 표준을 어떻게 지키게 했나 | 문서화만으론 안 지켜져서 grep 게이트 4종을 PR 조건으로 걸었음. 위반 3종 0 hit 유지 |
| 뺀 것 중 가장 아까운 건 | 차트 드로어. 다만 기준을 먼저 세워두고 지표 100% 중복을 확인해서 판단은 명확했음 |
| AI가 짠 코드 아닌가 | 맞다. 타이핑은 AI가 했고 나는 대안 비교·실측 검증·표준 정의·스코프 판단을 했다. 위 수치는 전부 내가 측정하거나 근거를 만든 것 |

---

## 출처

- [[3. 프로젝트/spx-agent/hdd/design.md]] — § 2.5 마트 DDL 최신 원본
- [[3. 프로젝트/spx-agent/hdd/specs/design/data-mart.md]] · [[3. 프로젝트/spx-agent/hdd/specs/requirements/data-mart.md]]
- [[3. 프로젝트/spx-agent/architecture.md]] — 마트 인터페이스 의무화
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] — 결정 이력 전체
- [[3. 프로젝트/spx-agent/references/audit-schema.md]] · [[3. 프로젝트/spx-agent/references/dashboard-query-inventory.md]]
- [[3. 프로젝트/spx-agent/references/model-token-chart-coverage.md]] · [[3. 프로젝트/spx-agent/references/external-connection-removal.md]] · [[3. 프로젝트/spx-agent/references/objects-charts-feasibility.md]]
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] — H-DASH-20 / 21 / 17 / 02
- [[0. Inbox/약식 부하 측정 + EXPLAIN ANALYZE 인덱스 hit 점검 결과.md]] — 응답시간 원본
- [[0. Inbox/마트 설계 결정 - 2026-05-14.md]] — 옵션 A~D 비교 원본
- [[0. Inbox/화면 설계 검토 v2 - KPI 개편 후.md]] · [[0. Inbox/대시보드 추가 위젯 후보 10개.md]]
- 코드: `spx-agent/dify-audit/prisma/audit/migrations/` (13개), `api/services/admin/dashboard_*.py` (9파일)

> [!warning] 문서 drift 주의 (인용 시)
> - `specs/{requirements,tasks}/data-mart.md`는 5/15 시점에서 멈춤 — `hdd/design.md`만 6/10 B안 반영
> - `references/dashboard-query-inventory.md`는 5/8 스냅샷이라 RBAC 무접두 표기 (실제는 `spx_`)
> - 문서상 "drill 11종"이지만 실제 소스는 **drill 12 + drawer 1 = 17라우트**
> - 성과 뱅크의 "3계층" 표현은 이 노트 기준 **2계층 4객체**가 정확

## 관련 노트

- [[포트폴리오/00 성과 뱅크.md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[3. 프로젝트/SPX-Agent 하네스 설계.md]]
