# Jenkins 배포 가이드 (194 서버)

> spx-agent 운영(dev) 서버 `192.168.10.194` 배포 절차. **태그 기준** 배포.
> 출처: 루트 `Jenkinsfile`, `docs/dev-merge-harness.md` §4·§7·§8.
> 작성: 2026-06-10

## 한눈에 보는 흐름

```
[로컬 dev] → master fast-forward 머지 → v1.x.y 태그 생성
   → master + 태그 같이 push
   → Jenkins "파라미터와 함께 빌드" (DEPLOY_REF=태그, DEPLOY_MODE 선택)
   → 194 서버에서 docker-compose 빌드·배포
   → 사후 점검
```

## 대상 환경

| 항목          | 값                                                                                              |
| ----------- | ---------------------------------------------------------------------------------------------- |
| 서버          | `192.168.10.194` (사내 dev 서버)                                                                   |
| 배포 경로       | `/opt/spx-agent`                                                                               |
| compose 파일  | `/opt/spx-agent/docker/docker-compose.yaml`                                                    |
| env 주입      | Jenkins credential `spx-deploy-env` (= `docker/.env.deploy` 내용) → `/opt/spx-agent/docker/.env` |
| Jenkins job | `spx-agent-deploy`                                                                             |

---

## 1단계 — master 머지 + 태그 생성/push

운영 배포는 **commit이 아니라 `v1.x.y` 태그 기준**. master를 fast-forward로 맞춘 뒤 태그를 만들고 **master와 태그를 같이 push**.

```powershell
git checkout master
git merge --ff-only dev
git tag -a v1.1.X -m "...요약..."     # 예: v1.1.19
git push origin master
git push origin v1.1.X                # 태그도 반드시 같이 push
```

- master는 항상 fast-forward 가능한 상태 유지
- 현재 태그 `v1.1.18`까지 존재 → 다음은 `v1.1.19` 식으로 증가

---

## 2단계 — Jenkins 빌드

1. Jenkins → `spx-agent-deploy` → **"파라미터와 함께 빌드"** 클릭
2. **`DEPLOY_REF`** 드롭다운에서 방금 만든 **태그(`v1.1.X`)를 명시적으로 선택/입력**
3. **`DEPLOY_MODE`** 선택
   - `recreate` — 코드만 반영, 다운타임 최소 (기본/일반)
   - `full-restart` — `down` 후 `up`, DB 마이그레이션·네트워크/볼륨 정의 변경 시

### 파이프라인 파라미터

| 파라미터 | 타입 | 값 | 설명 |
|----------|------|-----|------|
| `DEPLOY_REF` | gitParameter (PT_TAG) | `v1.x.y` 태그 | 배포할 태그 (내림차순 드롭다운) |
| `DEPLOY_MODE` | choice | `recreate` / `full-restart` | 코드만 / 전체 재기동 |

### 파이프라인 동작 (Jenkinsfile)

1. **Git Pull** — `git fetch --all --tags --prune` → `git checkout -f ${DEPLOY_REF}`
2. **Setup Env** — credential `spx-deploy-env` → `docker/.env` 복사
3. **Build** — `cd docker && docker-compose build`
4. **Deploy**
   - `full-restart`: `docker-compose down --remove-orphans` → `up -d`
   - `recreate`: `docker-compose up -d --force-recreate`

---

## 3단계 — 사후 점검 (194 서버)

```bash
cd /opt/spx-agent/docker
docker compose ps
docker compose logs --tail 50 api      | grep -iE "alembic|migration|error"
docker compose logs --tail 50 keycloak | grep -iE "imported|listening|error"
docker compose logs init_keycloak_realm_extras | tail -10
```

- 인증 스모크: 운영 사용자 1명으로 실제 로그인 → 콘솔 진입 → 권한 동작 확인

---

## 롤백

이전 안정 태그로 재배포:

```
Jenkins → spx-agent-deploy → 파라미터와 함께 빌드 → DEPLOY_REF = v1.1.X-1
```

DB 다운그레이드가 필요하면 `docker compose exec -T api alembic downgrade -1`
(⚠️ 일부 마이그레이션은 의도적으로 downgrade가 비어 있음 → 불가 시 백업 후 fresh restore)

---

## 알려진 함정

| 함정 | 증상 | 대응 |
|------|------|------|
| `DEPLOY_REF` 빈 값으로 빌드 | 옛 commit으로 빌드, 변경 미반영 | **항상 태그 명시 입력** |
| 태그만 만들고 master push 안 함 | 태그가 옛 commit을 가리킴 | `git push origin master` + 태그 **같이** |
| `spx-deploy-env` credential 구버전 | 배포 시 옛 `.env`(예: 옛 `KEYCLOAK_REALM`)로 덮어써져 mismatch 재발 | git의 `.env.deploy`로 credential 업데이트 |
| `KEYCLOAK_REALM` 변경 후 운영 DB rename 누락 | backend → keycloak `realm not found` 401 | `dev-merge-harness.md` §5.4 kcadm rename 절차 |

---

## 참조

- 루트 `Jenkinsfile` — 파이프라인 정의
- `docs/dev-merge-harness.md` §4(Push & 배포)·§5(사후 점검)·§7(롤백)·§8(함정)
- `spx-agent-docs/jenkins-deploy-setup.md` — Jenkins 셋업
- `spx-agent-docs/infra-overview.md` — 인프라 전반
