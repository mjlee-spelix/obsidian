---
tags: [프로젝트, dify, AI-Agent, HDD, delegation]
type: standard
date: 2026-05-19
last_updated: 2026-05-19
purpose: AI 에이전트(VSCode Claude / general-purpose agent / external Claudian)에 작업 위임 시 PROMPT.md 본문에 반드시 박아야 할 표준 가드
---

# 위임 프롬프트 표준 (Delegation Standard)

> 새 위임 PROMPT.md를 작성할 때마다 본 § 1~§ 5 가드 5종을 **본문에 박거나 포인터로 인용**할 것. 가드 빠진 위임은 5/15·5/18·5/19 학습이 재발할 위험.

## 배경

2026-05-15·05-18·05-19 누적 사고가 모두 **위임 시점에 가드가 없어 생긴 패턴**이었음:
- 5/15 audit rename 시점에 trigger 함수 재배포 누락 (자가 검증 거짓)
- 5/18 drift 1차/2차 — design.md 본 갱신만 박힌 채 마이그레이션 본문이 어긋남 (drift 미감지)
- 5/19 keycloak realm + dify-audit dist 사고 — `restart` 단독으로 새 정의 미반영 (컨테이너 명령 세트 부재)
- PowerShell 한글 인코딩 함정 (`Get-Content -Raw | docker exec -T` 패턴)

본 표준은 위 학습을 PROMPT.md에 강제로 박는 형태로 박제. 위임을 받는 AI가 가드를 인지한 채 작업 진행 → 동일 사고 재발 차단.

## § 1. drift 발견 시 작업 중단 의무

### 조항
위임 작업 중 **`hdd/design.md` ↔ 실제 DB DDL** 불일치(drift) 발견 시:
1. 작업 즉시 중단
2. 사용자(또는 위임 발주자)에게 drift 본문 보고 — design.md 쪽 / DB 쪽 어느 쪽이 진실인지 결정 요청
3. **자동 보정 금지** — 설계 의도 vs 실제 운영 어느 쪽을 정답으로 둘지 판단은 사용자 영역

### 시뮬레이션
- 작업 D(목업 데이터 보강) 진행 중 fixture INSERT 직전 `\d public.spx_mv_audit_enriched` 확인 → enriched view 컬럼 순서가 design.md와 다름 발견
- **가드 작동**: "design.md § 2.5.4 enriched view DDL에 `error_text` 다음에 `app_owner_dept_id`이지만 실제 DB는 순서 반대 — 작업 중단, 어느 쪽을 진실로 둘까요?" 보고 → 사용자 답변 받은 후 진행

### 검증
- 위임 보고서에 "drift 발견 0건" 또는 "drift 발견 N건, 모두 사용자 보고 후 결정 반영" 명시
- `scripts/check-mart-drift.ps1`(작성 예정) 또는 수동 `psql \d+` vs design.md grep diff 실행 결과 첨부

### 관련
- H-DOC-01 (design.md ↔ 마이그레이션 후속 작업 분리)
- `quality-criteria.md § 2 백엔드` drift 게이트

## § 2. 자가 검증 거짓 가능성 가드

### 조항
"완료" 보고 전에 **실제 출력으로 직접 확인**해야 함. 다음 중 1개 이상 첨부:
- `\dt` / `\d+ <object>` / `SELECT count(*) FROM ...` 출력
- `ls <파일경로>` / `cat <file>` 출력
- API endpoint hit 결과 (`curl ... | jq ...` 응답 본문)
- grep / `git status` / `git log -1` 출력

박은 문서 / 작성한 코드 / 적용한 마이그레이션이 **실제로 박혔는지**를 출력으로 입증. "박았습니다", "적용 완료" 단독 보고 금지.

### 시뮬레이션
- 작업 후 "defect-catalog.md에 H-MART-01 등록 완료" 보고만 박음
- **가드 작동**: 보고 검토자가 "출력 첨부 누락 → 자가 검증 거짓 가능성" 지적 → `grep -n "H-MART-01" hdd/defect-catalog.md` 출력 또는 해당 섹션 본문 인용 첨부 요구

### 검증
- 보고서의 각 작업 결과에 출력 인용 1건 이상
- 인용 없는 항목은 "검증 미완료"로 분류 → 위임 발주자가 추가 확인 또는 재작업 결정

### 관련
- 5/15·5/18 학습: "박았다"고 보고했으나 실제 잘못 박혀있어 후속 작업에서 발견된 사례 다수
- H-DASH-17 (외부 의존 명세 추정 함정) — 본 가드의 가족 결함

## § 3. 마이그레이션 본문 작성 표준 — idempotent 의무

### 조항
새 마이그레이션 작성 시 다음 패턴 의무:
- **`CREATE`**: `CREATE TABLE/VIEW/INDEX ... IF NOT EXISTS` 또는 `CREATE OR REPLACE VIEW/FUNCTION`
- **`DROP`**: `DROP ... IF EXISTS [CASCADE]`
- **RENAME / 스키마 이동**: `DO $$ BEGIN IF EXISTS (SELECT 1 FROM information_schema.tables WHERE ...) THEN EXECUTE 'ALTER TABLE ... RENAME TO ...' END IF; END $$;` 조건부 블록
- **데이터 이전**: `IF EXISTS` 조건부 데이터 이전 패턴
- **MView refresh**: `DO $$ BEGIN IF EXISTS (SELECT 1 FROM pg_matviews WHERE ...) THEN REFRESH MATERIALIZED VIEW ... END IF; END $$;`

목적: Prisma `migrate dev` Shadow DB 재생 + 운영-신규 환경 차이 둘 다 흡수 (H-INFRA-01 방어).

### 시뮬레이션
- 새 마이그레이션에 `CREATE MATERIALIZED VIEW public.spx_v_xxx AS ...` 박음 → Shadow DB 재생 시 이미 존재해 에러
- **가드 작동**: 작성 시점에 `CREATE MATERIALIZED VIEW` → `DROP MATERIALIZED VIEW IF EXISTS ... CASCADE; CREATE MATERIALIZED VIEW ...` 패턴 박기 의무 인지

### 검증
- 새 마이그레이션 파일에 `IF NOT EXISTS` / `IF EXISTS` / `OR REPLACE` / `DO $$` 중 1개 이상 등장
- `docker compose down -v && docker compose up -d` 깡통 부팅 검증 통과

### 관련
- H-INFRA-01 (Shadow DB 부채)
- 5/18 갈래 1 + 갈래 2 합성 솔루션 (SESSION_HISTORY)
- `dify-audit/prisma/audit/migrations/20260518100000_actor_id_non_uuid_safe/migration.sql` 표준 예시

## § 4. 컨테이너 변경 시 명령 세트 — restart 단독 금지

### 조항
다음 변경 후엔 `restart` 단독 금지, **`build + up -d --force-recreate` 의무**:
- **entrypoint script 변경** → `docker compose build <service> + up -d --force-recreate <service>`
- **docker-compose.yaml volume/env 변경** → `docker compose up -d --force-recreate <service>` (build 불필요 시)
- **src 변경 (특히 raw query 박힌 collectors)** → `docker compose build <service> + up -d --force-recreate <service>`

동료에게 안내 시 슬랙 메시지에 **3종 세트** (`git pull → docker compose build → docker compose up -d --force-recreate`) 박기.

### 시뮬레이션
- dify-audit collector raw query 갱신(`accounts` → `spx_accounts`) 후 동료에게 `restart` 안내
- **가드 작동**: 옛 dist 박힌 채 restart → `relation "accounts" does not exist` 폭발. 안내 시점에 3종 세트 박기로 사전 차단

### 검증
- 위임 작업이 컨테이너 변경 동반 시 보고서에 "build + recreate 적용" 출력 첨부 (`docker compose ps` 또는 `docker inspect ... --format '{{.Created}}'`)
- 동료 안내 슬랙 본문이 3종 세트 형식 따르는지 확인

### 관련
- H-INFRA-02 (compose mount drift)
- H-INFRA-03 (dist 빌드 캐시 함정)
- H-ENV-02 (api/worker build 지시 부재)

## § 5. PowerShell 한글 인코딩 함정 회피

### 조항
한글 SQL 본문 또는 한글 포함 파일을 컨테이너에 전달 시:
- **금지**: `Get-Content -Raw <한글-SQL> | docker exec -T <container> psql ...` (UTF-16 LE BOM 또는 cp949 변환으로 깨짐)
- **표준 패턴**:
  1. 호스트에서 SQL 파일 작성 (UTF-8 BOM **없이** — `Out-File -Encoding utf8` 시 PS 5.1은 BOM 박음, `[System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($false))` 권장)
  2. `docker compose cp <local-sql> <container>:<path>` 로 파일 전달
  3. `docker compose exec <container> psql -U postgres -d dify -f <path>` 로 실행
- **인코딩 검증**: `file <path>` 출력에 `UTF-8 Unicode text` (no BOM 표시 없음) 확인

### 시뮬레이션
- 한글 코멘트 박힌 SQL을 PowerShell 5.1에서 `Get-Content -Raw | docker exec -T ... psql`로 실행
- **가드 작동**: 한글 인코딩 깨져 SQL parser 에러 → 표준 패턴 (`cp + exec -f`) 으로 우회

### 검증
- 위임 보고서에 한글 SQL 실행 결과(`COMMIT` 또는 `INSERT 0 N` 출력) 첨부
- `psql ... -f <path>` 패턴 사용 (Get-Content piping 미사용) 확인

### 관련
- 5/15·5/18·5/19 누적 학습 (PowerShell 인코딩/escape 함정)
- 지식노트 [[4. 지식노트/PowerShell - 한글 인코딩과 docker exec piping 함정]] (작성 예정)

---

## PROMPT.md 본문에 박는 형태 (템플릿)

새 위임 PROMPT.md 본문 끝에 다음 섹션 박기 의무:

```markdown
## 위임 표준 가드 (필수 인지)

본 위임은 `hdd/delegation-standard.md`의 § 1~§ 5 가드 5종 적용 대상:

1. **drift 발견 시 작업 중단** (§ 1) — design.md ↔ DB 불일치 시 자동 보정 금지
2. **자가 검증 거짓 가능성** (§ 2) — "완료" 보고 전 실제 출력 첨부 의무
3. **마이그레이션 idempotent** (§ 3) — IF NOT EXISTS / DO $$ 조건부 패턴 의무
4. **컨테이너 변경 시 build + recreate** (§ 4) — restart 단독 금지
5. **PowerShell 한글 인코딩** (§ 5) — docker cp + psql -f 패턴

상세: `.claude/docs/hdd/delegation-standard.md` 참조
```

위 섹션이 빠진 PROMPT.md는 본 표준 미준수. 위임 발주자는 본 섹션 박힘 여부 사전 확인 의무.

## 검증 (본 표준 자체)

본 표준이 박힌 후 첫 위임(예: 5/19 작성된 마트 부하 테스트 목업 데이터 보강 PROMPT.md)에 § 1~§ 5 가드 인용 박힘 확인 — 박혔다면 본 표준 정상 작동.

## 관련 노트

- `defect-catalog.md` — H-MART-01 / H-INFRA-01·02·03 / H-DOC-01 (본 표준 가드의 본문 결함)
- `quality-criteria.md` § 2 백엔드 drift 게이트
- `CLAUDE.md` 진입점 지도 — 위임 시 첫 진입점
- `SESSION_HISTORY.md` 2026-05-15 / 05-18 / 05-19 entry — 본 표준의 학습 출처
