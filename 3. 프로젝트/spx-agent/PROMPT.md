# 위임 프롬프트 — dev dify-audit 재빌드(B안 적용) + dify_dev 검증

> 작성일: 2026-06-10
> 작업 위치: `C:\Users\Administrator\Projects\spx-agent\` + 194 서버(dify-dev, SSH)
> 가드: `hdd/delegation-standard.md` § 2(자가검증 출력)·§ 4(컨테이너 build+recreate)
> 선행 컨텍스트: `SESSION_HISTORY § 2026-06-10 194 DB 손상 결말 + 인프라 토폴로지`

---

## 배경 (왜 이 작업)

- 194 prod-infra DB 손상 후 재구성: `dify`(빈) → `dify_prod`(prod 원천) + `dify_dev`(dev 원천+audit). 모두 한 postgres(`spx-prod-infra-db_postgres-1`, **194:15432**).
- **dify-audit는 dev에만**(`spx-dev-dify-audit-1` → `dify_dev`). 단 **현재 dev 이미지는 B안 변경 전(stale)** — `dify_dev`에 마이그레이션 `20260610`(B안 UNION) 없음, model_tokens가 옛 정의(`FROM spx_mv_audit_enriched`).
- → **dev dify-audit를 KAN-29(B안 포함)로 재빌드**하면 마이그레이션 적용 + collector process_data 파싱 + self-heal 경화(DR-6) 반영됨. 그 후 dify_dev에서 B안 검증(시드/차트) 가능.
- ⚠️ **DEV 환경만** — `dify_prod`/`dify`(15432) 건드리지 말 것. prod audit 배포는 별도(범위 밖).

---

## 작업 0 — 재빌드 선행 확인 (코드가 서버 빌드에 닿는지)

> 재빌드는 **194 서버에서** 일어남(로컬 빌드 아님). 우리 B안 코드가 **git → 194 서버**로 닿아야 빌드에 반영됨. 아래 확인 후 작업 1. **하나라도 안 되면 중단+보고.**

### 0-1. 로컬 — B안이 git(KAN-29)에 올라갔나
- [x] `git branch --show-current` → `KAN-29-admin-dashboard`
- [x] `git status` → 미커밋 변경 없나 (collector/마이그레이션/서비스/프론트)
- [x] 핵심 파일이 브랜치에 커밋돼 있나:
  - `dify-audit/prisma/audit/migrations/20260610000000_rewrite_model_tokens_daily_union/migration.sql`
  - `dify-audit/src/lib/collectors/workflow-nodes.ts` (process_data 파싱)
  - `dify-audit/scripts/entrypoint.sh` (self-heal 리스트에 20260610 + DR-6 경화)
  - `api/services/admin/dashboard_drill_calls_service.py` (model-call-share repoint) / `api/models/mart.py`(타입 수정)
- [x] `git log origin/KAN-29-admin-dashboard..HEAD --oneline` → **출력 0줄**(전부 push됨). 있으면 **push 필요**.

### 0-2. dev 배포 방식 확인
- [ ] dev 스택(`spx-dev-*`)이 **Jenkins** 배포냐 **수동 compose**냐.
- [ ] dev compose 파일 위치 + `dify-audit` 서비스 정의 + **빌드 컨텍스트/소스 브랜치**(KAN-29 반영되나).
- [ ] (Jenkins면) dev 배포 job + `DEPLOY_REF` 지정 방법 — 참고 [[3. 프로젝트/spx-agent-docs/docs/references/jenkins-deploy-194.md]].

### 0-3. 194 서버가 KAN-29 코드를 받았나
- [ ] **(수동)** 194 SSH → repo 위치 → `git fetch && git status` → KAN-29 최신인지 → 아니면 `git checkout KAN-29-admin-dashboard && git pull`.
- [ ] **(Jenkins)** 배포 시 KAN-29(or 해당 태그)로 체크아웃되는 구조인지.
- [ ] 빌드 컨텍스트가 **서버의 그 repo 경로**를 가리키는지 (빌드 시 최신 코드 반영 확인).

### 0-4. 선행 게이트
- [ ] **B안이 git+서버에 닿음 확인되면 작업 1 진행.** 안 닿았으면(로컬에만 있음/미push/서버 미pull) → **먼저 push/pull 해결**, 안 그러면 재빌드해도 stale 그대로.

## 작업 1 — dev dify-audit 재빌드 + 재시작

- [ ] **dev 스택 compose/소스 확인** — `spx-dev-*` 스택이 어느 compose 파일 + 어느 브랜치/이미지로 뜨는지. KAN-29(B안)로 빌드되는지 확인. (Jenkins 배포면 그 경로, 수동이면 직접)
- [ ] **현재 stale 확인**(재현) — `15432/dify_dev`에서:
  ```sql
  SELECT migration_name FROM _prisma_migrations WHERE migration_name LIKE '%model_tokens%';  -- 없음(B안 전)
  SELECT left(definition,150) FROM pg_matviews WHERE matviewname='spx_mv_model_tokens_daily';  -- FROM spx_mv_audit_enriched(옛)
  ```
- [ ] **재빌드** — `docker compose -f <dev compose> build --no-cache dify-audit` (⚠️ **`--no-cache` 필수**, H-INFRA-03: 일반 build는 캐시로 구 소스 컴파일).
- [ ] **재시작** — `docker compose -f <dev compose> up -d --force-recreate dify-audit`.
- [ ] **entrypoint 로그 확인** — `docker logs --tail 100 spx-dev-dify-audit-1` → 마이그레이션 `20260610_rewrite_model_tokens_daily_union` **적용(applied)** + self-heal 미실행(or 정상) + collector run 정상.

## 작업 2 — B안 적용 검증 (dify_dev)

- [ ] `15432/dify_dev`에서:
  - [ ] `20260610...union` 마이그레이션 존재
  - [ ] model_tokens 정의가 **UNION**(`spx_audit_events` 직접 + `sub` 서브쿼리, `message_send` 비챗플로우 + `workflow_node_execute`)
  - [ ] OWNER `audit_writer`, UNIQUE 인덱스 존재
- [ ] **collector process_data 파싱 확인** — 재빌드 후 새 수집분: `SELECT model_provider_d, model_id_d, total_tokens_d FROM spx_audit_events WHERE action='workflow_node_execute' AND model_id_d IS NOT NULL LIMIT 5;` → 모델노드에 채워지나. (기존 4863건 옛 수집분은 모델키 없음 — 정상)

## 작업 3 — 3.6 mock 시드 (dify_dev, 대시보드 가시화)

> 상세: `hdd/specs/tasks/workflow-model-classification.md` 3.6단계(MS-1~10). DEV라 안전.

- [ ] **현황(MS-1)** — `15432/dify_dev`에서 `wn_`/`ms_` 행수, `spx_mv_model_tokens_daily` 현재 모델 목록(baseline).
- [ ] **시드(MS-5)** — `api/scripts/seed/audit_mock.sql`의 **S1~S6(`wn_`/`ms_`) 부분만** dify_dev `spx_audit_events`에 INSERT. 맨 위 `DELETE ... id LIKE 'wn_%'/'ms_%'`로 멱등.
- [ ] **REFRESH(MS-6)** — `REFRESH MATERIALIZED VIEW CONCURRENTLY public.spx_mv_model_tokens_daily;`
- [ ] **검증(MS-8)** — 마트에 워크플로우/챗플로우 모델 등장 + S3 챗플로우 토큰합 == 노드합(24,000) + '미분류' 0.

## 작업 4 — dev 대시보드 라이브 확인 (5단계 시각 검증)

- [ ] dev 대시보드(`spx-dev-web`)에서 **모델별 토큰** + **모델별 호출 점유** 차트에 워크플로우/챗플로우 모델 등장 + 단위 툴팁.
- [ ] ⚠️ **로컬 라벨 확인** — `dify_dev` provider가 namespaced(`langgenius/ollama/ollama`, `yangyaofei/vllm/vllm`)면 **"(로컬)" 안 뜰 것**(LOCAL_PROVIDERS bare 매칭 실패) = **알려진 버그**. 뜨는지/안 뜨는지 기록만(fix는 별도 트랙). 확인: `SELECT DISTINCT provider_name FROM provider_models;`
- [ ] (선택) task 27 provider 포맷 dify_dev로 재확인.

---

## 가드 / 제약

- **§ 2 자가검증**: 각 작업 **실제 출력 첨부**(마이그레이션 로그·쿼리 결과). "적용됐다" 추측 금지.
- **§ 4 컨테이너**: `--no-cache` build + `--force-recreate` 필수. `restart` 단독 금지.
- ⛔ **DEV만** — `dify_prod`/`dify`(15432의 다른 DB) 쓰기/DDL 절대 금지. dify_dev만.
- ⛔ **`down -v` 금지** — 공유 postgres라 다른 DB까지 영향.
- self-heal: 재빌드 이미지엔 DR-6 경화(ON_ERROR_STOP 제거) 포함됨 — 적용 후 마트 정상 생성 수동 확인.

## 범위 밖 (별도 트랙)

- **prod(dify_prod) audit 배포** — 같은 이미지 + env `dify_prod`로 dify-audit **인스턴스 추가**(prod 스택 compose에 서비스 추가). dev 컨테이너 재활용 불가. dev 검증 통과 후 진행.
- **로컬 라벨 정규화 fix** — provider namespaced(마지막 세그먼트 추출) + `vllm` 추가. 조사 완료 → `references/local-provider-classification.md` 참조. 코드+테스트(namespaced 입력) 별도.
- **27 중첩 노드(28)** — 실 중첩 워크플로우 데이터 필요.
- 완료 후 `SESSION_HISTORY`에 결과 박제.
