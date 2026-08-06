---
tags: [Docker, git, Windows, 개발, CS, CAND]
date: 2026-05-07
---
# Docker — Windows 호스트 git checkout 시 워킹트리 깨짐

## 핵심

컨테이너가 코드 디렉토리를 bind mount로 잡은 채 `git checkout`/브랜치 전환을 하면, **Windows 파일 시스템의 file-in-use 락 + git의 파일 정리 동작**이 충돌해 새 브랜치에 없는 파일이 **빈 디렉토리 형태**로 워킹트리에 잔존. 다음 컨테이너 부팅 시 호스트의 깨진 형태(빈 디렉토리)를 그대로 마운트 → Python `.py` import 불가 → 컨테이너 무한 재시작.

## 발견 사례 (2026-05-07)

증상: API 컨테이너 health: starting 무한 반복. 로그:
```
ERROR [flask_migrate] Error: Can't locate revision identified by 'c1d2e3f4a5b6'
```

진단 흐름:
1. `docker ps` → API만 5초 전 재시작 (다른 컨테이너 정상)
2. `docker logs api` → flask_migrate가 alembic revision 못 찾음
3. `psql alembic_version` → DB는 c1d2 정상 박혀있음
4. 호스트 `versions/` 폴더 확인 → 그 마이그레이션 파일이 `Mode: d-----` (**빈 디렉토리**)로 깨져있음. `Get-ChildItem -Filter '*.py'`엔 잡히지만 안에 0건
5. `git ls-tree origin/feat/rbac:api/migrations/versions/` → 그 파일은 git blob으로 정상 존재 (다른 브랜치)

추정 메커니즘:
- 작업 중 `feat/rbac` → `KAN-29-...` 브랜치로 checkout
- docker 컨테이너가 `api/migrations/versions/`를 bind mount로 잡고 있음
- Windows file-in-use 때문에 git이 파일을 깨끗이 삭제 못 함
- entry는 사라지지만 빈 디렉토리만 잔존
- 다음 컨테이너 부팅 시 호스트의 깨진 형태(빈 디렉토리)를 그대로 마운트
- Python `.py` import 불가 → flask가 그 revision 모듈을 못 읽음

## 복구

```powershell
docker compose stop api worker worker_beat
Remove-Item <빈 디렉토리 경로> -Recurse -Force
git checkout origin/<src-br> -- <마이그레이션 파일 path>
docker compose start api worker worker_beat
```

## 방어

### 1. 권장 — 브랜치 전환 전 컨테이너 stop

```powershell
docker compose stop api worker worker_beat
git checkout <other-br>
# 작업 후
git checkout <original-br>
docker compose start api worker worker_beat
```

### 2. 사후 검증 명령

```powershell
Get-ChildItem api\migrations\versions -Directory | Where-Object Name -Match '\.py$'
```

결과 0건이면 OK. 출력된 디렉토리는 깨진 형태.

### 3. 컨테이너 안 검증

```bash
docker exec <api> sh -c "cd /app/api && flask db heads"
```

head revision이 출력되면 마이그레이션 정상 인식.

## 같은 가족 함정

- H-ENV-01: Linux .venv lib64 symlink → Windows reparse point 깨짐 → BuildKit context 스캔 실패
- H-ENV-02: api/worker 서비스 build 미지정 → 호스트 코드 변경 미반영
- → 모두 *"Windows 호스트 ↔ Linux 컨테이너 ↔ 우리 코드"* 3자 동시성에서 발생

## 검증 필요 (CAND 단계)

- 1차 발생만 검증됨 (2026-05-07)
- 다른 파일 유형(.json/.yaml/.ts)에서도 재현되는지 미확인
- Linux/macOS 호스트에서도 발생하는지 미확인 — Windows file-in-use 동작이 핵심 트리거인지 검증 필요
- 1건 추가 발생 시 정식 H-ENV-04 승격

## 관련 노트

- [[4. 지식노트/Windows - 심볼릭 링크로 폴더 동기화]]
- [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]]

## 관련 결함

- spx-agent `defect-catalog.md` `CAND-git-checkout-with-running-container`
