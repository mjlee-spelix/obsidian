---
tags: [개발, PostgreSQL, Prisma, DB, CS]
date: 2026-05-15
related:
  - "[[4. 지식노트/PostgreSQL - search_path와 schema 네임스페이스]]"
  - "[[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]]"
  - "[[3. 프로젝트/spx-agent/SESSION_HISTORY.md]]"
---

# Prisma migrate — Shadow DB와 entrypoint 의존성 부채

## 문제 상황

`pnpm prisma migrate dev`로 새 migration 만들려고 하면 다음 에러 발생:

```
Shadow DB에서 이전 마이그레이션 재생 실패
```

원인: 우리 환경의 **migration들이 entrypoint script가 미리 깔아준 환경(audit schema)에 의존**해서 작성됨. Shadow DB(빈 PG)에선 entrypoint가 안 돌아 → 첫 migration부터 `audit schema does not exist` 실패 → 새 migration 못 만듦

발견 컨텍스트: SPX-Agent dify-audit에서 audit schema → public schema rename 작업 후 (5/15) Layer 1 MView 추가하려는데 Shadow DB 검증에서 막힘.

## 핵심 인사이트

### 1. Prisma migrate dev의 7단계
```
1. schema.prisma 읽기
2. 이전 schema와 비교 (diff)
3. 🚨 Shadow DB에 과거 모든 migration 재생 + 새 변경 적용 (drift 검증)
4. SQL migration 파일 생성
5. 실제 DB에 새 migration 적용
6. _prisma_migrations 테이블에 기록
7. Prisma Client 재생성
```

### 2. Shadow DB의 정체
- 매번 **drop → recreate**되는 빈 PostgreSQL DB (`<원본>_shadow`)
- 모든 과거 migration을 처음부터 순서대로 재생
- 목적: "이 migration 시퀀스가 빈 DB에서도 정상 동작하나?" 검증
- 사용 케이스: 신규 개발자 환경 셋업 / CI / 운영 첫 셋업의 안전성

### 3. entrypoint script vs Migration 역할 차이

| | entrypoint script | Prisma Migration |
|---|---|---|
| **언제 실행** | 컨테이너 부팅마다 (반복) | 한 번만 (이력 관리) |
| **무엇을** | DB 인프라 셋업 (schema, role, 권한, trigger) | 데이터/스키마 진화 (테이블, 컬럼, 인덱스) |
| **패턴** | 멱등 (`IF NOT EXISTS`, `CREATE OR REPLACE`) | 순차 (한 번만) |
| **데이터 영향** | 거의 없음 | 있음 |

→ 분리는 본질이 다르니 합리적. **단 두 영역 의존성이 잘못 박히면 Shadow DB에서 사고**

### 4. 우리 사고의 본질
- entrypoint가 audit schema 깔아줌
- migration은 "audit schema 미리 있다"고 가정한 채로 박힘 (`CREATE TABLE audit.X`, `ALTER TABLE audit.X SET SCHEMA public` 등)
- Shadow DB는 entrypoint 안 거치고 빈 PG로 시작 → migration의 가정이 깨짐 → 실패

### 5. 우회 패턴 (사용된 임시방편)
`migrate dev` 자체를 안 쓰고:
1. `mkdir -p prisma/.../migrations/<timestamp>_<name>` (수동 폴더)
2. `migration.sql` 안에 raw SQL 직접 작성
3. `psql`로 실제 DB에 직접 실행
4. `prisma migrate resolve --applied <name>`으로 "이미 적용됨" 등록

→ Shadow DB 검증 단계 완전 우회. **Prisma migration은 형식만 빌리고 실제 일은 수동**

### 6. 부채 누적
우회로 만들어진 migration들도 Shadow DB에서 못 통과 → 다음 작업 시 또 우회 → 누적. 결과:
- 운영/본인 환경 (entrypoint 거침): 동작 OK
- 신규 환경 셋업 / CI (entrypoint 안 거침): 깨짐
- migration history와 실제 DB 정합성 점점 무너짐

## 해결 방향 3가지

### 방향 A — Migration 자체완결화 (idempotent 패턴)

기존 migration에 멱등 패턴 추가:
```sql
CREATE SCHEMA IF NOT EXISTS audit;
ALTER TABLE IF EXISTS audit.X SET SCHEMA public;
```

또는 audit schema 생성 자체를 migration #0으로 추가 (entrypoint 책임 일부 흡수)

- **작업량**: 1~2시간
- **위험**: 낮음 (과거 migration 수정만)
- **효과**: Shadow DB 통과. 향후 작업 정상 흐름. 신규 환경 셋업 가능
- **언제**: 다음 안정화 시점 (작업 한 사이클 끝나고)

### 방향 B — Migration squash

과거 migration 전부 지우고 **현재 DB 상태를 그대로 만드는 단일 init migration**으로 통합:
```
v1: spx_audit_events + Generated Column 8 + 인덱스 + Layer 1 MView + ... (한 번에)
```

- **작업량**: 반나절
- **위험**: 중 (history 손실)
- **효과**: 가장 깨끗
- **언제**: 마트 작업 다 끝나고 안정화된 후

### 방향 C — 부채 수용 (현 상태 유지)

`migrate dev` 우회 패턴 계속 사용. migration history 깨진 채로 운영. 신규 환경 셋업은 별도 init script로 처리

- **작업량**: 0
- **위험**: 누적될수록 깊어짐 (6개월 후 신규 개발자가 환경 셋업 못 함)
- **언제**: 단기 진행 (현재 상태)

## 우리 환경 추천 진행

| 시점 | 방향 | 이유 |
|---|---|---|
| **현재 (Layer 1·2 작업 중)** | C — 부채 수용 | 작업 흐름 끊지 않기 위해 |
| **마트 안정화 시점** | A — Idempotent 추가 | 작업량 적고 위험 낮음 |
| **장기** | B — Squash 검토 | 마트 작업 다 끝나고 |

## 학습 정리

- DB 마이그레이션 도구는 **"빈 DB에서도 모든 migration이 자체완결로 통과해야 한다"**는 가정에 깔려 있음
- entrypoint script로 환경 깔아주는 패턴은 **운영 편의를 위한 것**이지 migration 무결성을 위한 게 아님
- 두 영역의 책임 분리는 합리적이지만 **migration이 entrypoint에 암묵적 의존**하면 무결성 깨짐
- Shadow DB 실패 시 우회는 가능하지만 **부채로 누적**됨 — 명시적으로 인지 + 정리 시점 박아야 함

## 발견 컨텍스트

- 5/15 SPX-Agent dify-audit audit schema → public schema rename 작업
- 직접 원인: `setup_audit_schema.sql`가 audit schema 생성을 entrypoint에서 담당 + migration들이 audit schema 미리 있다고 가정
- 추가 사고: 5/15 audit rename + collector P0 Generated Column 마이그레이션 모두 같은 우회 패턴으로 만들어짐 (이미 부채 누적 중)
- 발견 시점: Layer 1 MView 작업하려고 `migrate dev` 박았더니 또 같은 에러 → 우회 패턴 또 반복하면서 의문 시작

## 관련 노트

- [[4. 지식노트/PostgreSQL - search_path와 schema 네임스페이스]] — schema 개념 기초
- [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]] — entrypoint 컨테이너 패턴
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] — 5/15 사고 entry
