---
tags: [Docker, 개발, 백엔드, dify]
date: 2026-05-04
---
# Docker - 호스트 코드를 컨테이너에 반영하는 패턴

## 핵심

> 컨테이너 안에서 도는 코드는 **이미지에 박힌 빌드 시점 스냅샷**. 호스트에서 코드를 수정해도 컨테이너에는 자동 반영되지 않음.
>
> 반영 방법은 크게 두 가지:
> 1. **빌드 패턴** — 이미지를 다시 만들어 새 코드를 박음 (5~10분, 운영용)
> 2. **마운트 패턴** — 호스트 폴더를 컨테이너에 "연결"해서 즉시 반영 (개발용)

각각 docker-compose.yaml의 `build:` 또는 `volumes:` 지시로 표현. **두 패턴 혼용 가능** (`docker-compose.override.yaml`로 환경 분리).

## 상세

### 1. docker-compose.yaml의 `image:` vs `build:`

```yaml
# 패턴 A — 레지스트리 이미지 그대로 사용 (코드는 이미 박혀있음)
api:
  image: langgenius/dify-api:1.13.3

# 패턴 B — 호스트 Dockerfile로 직접 빌드 (호스트 코드가 이미지로)
api:
  build:
    context: ../api
    dockerfile: Dockerfile
```

- `build:` 있으면 `docker compose build` 시점에 호스트 Dockerfile로 빌드 → 그 결과 이미지 사용
- `build:` 없으면 `docker compose build api` = `No services to build` (no-op). 레지스트리에서 이미지 받아씀.
- **혼란 포인트**: Dify의 `docker-compose.yaml`에서 web은 `build:` 있는데 api는 없을 수 있음. 사용자가 의도적으로 추가했거나 안 추가한 결과.

### 2. Volume mount — 호스트 폴더를 컨테이너에 "연결"

```yaml
volumes:
  - ../api:/app/api
  #   ^^^^^^      ^^^^^^^^^
  #   호스트 경로  컨테이너 안 경로 (호스트로 덮어씀)
```

비유:
```
호스트:                    컨테이너:
~/projects/spx-agent/      /app/
└── api/                   └── api/    ← 호스트 ../api/ 가 여기 통째로 보임
    └── controllers/           └── controllers/
        └── admin/                 └── admin/
```

→ 컨테이너 안 프로세스가 `/app/api/controllers/admin/dashboard.py`를 열면 **사실은 호스트 파일**. 호스트에서 한 줄 고치면 즉시 반영.

#### 부분 mount vs 통째 mount

```yaml
# 부분 — 특정 파일/폴더만 patch
- ../api/controllers/console/admin:/app/api/controllers/console/admin
- ../api/services/admin:/app/api/services/admin

# 통째 — api 폴더 전체 덮어씀
- ../api:/app/api
```

- 부분: 통제 정확, 새 폴더 추가 시 mount도 추가 필요 (귀찮)
- 통째: 한 번 설정으로 모든 변경 반영, 단 환경 차이(`.venv` 같은) 문제 생김 → **마스킹 필요**

### 3. `docker-compose.override.yaml` — 자동 머지로 dev 환경 분리

docker compose는 같은 폴더에 `docker-compose.override.yaml`이 있으면 **자동으로 머지**해서 적용.

```
docker/
├── docker-compose.yaml          ← 기본 (운영 가정)
└── docker-compose.override.yaml ← 추가 (dev 가정)

= 둘 합쳐진 가상의 단일 설정으로 동작
```

**가치**: 원본 `docker-compose.yaml` 무수정. 팀에서 base는 공유, 각자 dev 설정만 override로 따로.

```yaml
# docker-compose.override.yaml
services:
  api:
    build:                              # base에 없는 build 추가
      context: ../api
      dockerfile: Dockerfile
    volumes:                            # 호스트 코드 mount 추가
      - ../api:/app/api
      - api_venv:/app/api/.venv         # 마스킹 (다음 섹션)

volumes:
  api_venv:
```

### 4. Named volume / Anonymous volume — 마스킹 패턴

#### 문제: 통째 mount하면 호스트의 모든 파일이 컨테이너에 보임

호스트 `.venv`는 OS 환경(예: Windows)에 맞춰 만들어졌는데, 컨테이너는 다른 OS(Linux). 호스트 `.venv`가 컨테이너에 그대로 보이면 Python 인터프리터 경로 등이 안 맞아서 깨짐.

#### 해결: 그 경로만 호스트 overlay 차단

```yaml
volumes:
  - ../api:/app/api               # 호스트 폴더 통째 덮음
  - api_venv:/app/api/.venv       # 단, .venv는 named volume으로 별도 보호
  - /app/api/__pycache__          # 단, __pycache__는 anonymous volume으로 호스트 차단

volumes:
  api_venv:                        # named volume 정의
```

- **Named volume** (`api_venv:`): 이름 있는 volume. 컨테이너 자체 파일로 채움 (이미지 빌드 시점의 .venv 내용). 명시적 이름이라 `docker volume rm`으로 타겟 삭제 가능.
- **Anonymous volume** (`/app/api/__pycache__`): 이름 없는 volume. 같은 효과 — 호스트 overlay 차단. 단 식별이 어려워 wipe하기 까다로움.

#### 언제 Named, 언제 Anonymous?
- **재생성이 필요할 가능성 있음** (예: pyproject.toml 변경 시 .venv 다시 채워야 함) → **Named** (타겟 삭제 가능)
- **컨테이너 라이프사이클 동안만 유효하면 됨** (예: 캐시) → **Anonymous** (간편)

### 5. 호스트 `.venv` vs 컨테이너 `.venv` — OS 환경 차이

```
호스트 (Windows):              컨테이너 (Linux):
api/.venv/                     /app/api/.venv/
├── Lib/                       ├── lib/                  ← 대소문자 다름
│   └── site-packages/         │   └── python3.12/site-packages/
└── Scripts/python.exe         └── bin/python            ← 다른 실행파일
```

→ **호스트 .venv를 컨테이너에 그대로 mount하면 Python 자체가 안 돌아감**. 마스킹 필수.

### 6. 의존성 변경 시 .venv 갱신

호스트 `pyproject.toml`에 새 패키지 추가 시:
- 호스트 `uv sync` 또는 `pip install` → 호스트 `.venv` 갱신 ✅
- 컨테이너 `.venv`는 이미지 빌드 시점에 박힌 거라 **자동 갱신 안 됨**
- 필요한 경우: 이미지 재빌드 + named volume wipe + 컨테이너 재기동

```bash
# .venv 새로고침 절차
docker compose down api
docker volume rm <project>_api_venv  # named volume 삭제
docker compose build api             # 새 이미지 (호스트 pyproject.toml 반영)
docker compose up -d api             # named volume 재생성 (새 .venv 채움)
```

> **함정**: 단순 `docker compose down api && up -d api`는 named volume 보존 → 새 이미지 빌드해도 .venv는 옛날 내용. 반드시 volume 삭제까지.

### 7. BuildKit 컨텍스트 스캔과 `.dockerignore`

`docker compose build`는 컨텍스트(Dockerfile 주변 폴더) 전체를 BuildKit에 전송. **BuildKit은 모든 파일을 stat한 후 `.dockerignore`를 적용**하므로:

- 깨진 symlink가 있으면 `.dockerignore`에 적혀있어도 stat 단계에서 폭발
- 예: Linux symlink가 Windows 호스트에서 깨진 reparse point로 보임 → 빌드 실패

#### 해결: 빌드 전 정리 wrapper

```powershell
# scripts/spx-build.ps1
# 1) 깨진 symlink 정리
.\scripts\spx-clean-broken-symlinks.ps1
# 2) docker compose build (... cd $dockerDir)
docker compose build @args
```

(SPX-Agent 프로젝트의 `H-ENV-01` 결함과 동일 사례.)

### 8. Dev vs Prod 모드 — Web도 같은 원리

| 모드 | 코드 변경 반영 | 설정 |
|------|------|------|
| Prod | 재빌드 필요 (5~10분) | image 박힌 산출물 서빙 |
| Dev | 즉시 반영 (hot reload) | volume mount + dev 서버 (`pnpm dev`, `flask run --debug`) |

기본 `docker compose up`만 하면 prod 모드로 도는 구성이 흔함. 코드 자주 바꾸는 개발 단계에선 mount 패턴 도입 가치 큼.

## 자주 만나는 함정 정리

| 증상 | 원인 | 해결 |
|------|------|------|
| `docker compose build api`가 `No services to build` | api 서비스에 `build:` 지시 없음 | override.yaml에 `build:` 추가 또는 base 수정 |
| 호스트 코드 변경했는데 컨테이너 안 바뀜 | mount 없거나 stale 이미지 | mount 추가 또는 재빌드 |
| `ImportError: No module named 'X'` | 호스트 pyproject에 추가했는데 컨테이너 .venv에 X 없음 | 이미지 재빌드 + named volume wipe |
| 파이썬 인터프리터 자체 안 돎 | 호스트 .venv (다른 OS)가 컨테이너에 mount됨 | named/anonymous volume으로 .venv 마스킹 |
| 빌드 시작도 못하고 `error from sender: open ...lib64` | 깨진 Linux symlink가 Windows에서 BuildKit stat 실패 | symlink 정리 wrapper 사용 |
| `docker compose down -v` 했더니 다른 데이터까지 사라짐 | `-v`는 프로젝트 모든 volume 삭제 | named volume만 타겟해서 `docker volume rm` |

## SPX-Agent 적용 사례

- `docker/docker-compose.override.yaml` — api/worker에 build + 통째 mount + named volume venv 마스킹
- `scripts/spx-build.ps1` / `.sh` — 빌드 전 깨진 symlink 정리 wrapper
- 결함 등록: [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|H-ENV-01]] (symlink), [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|H-ENV-02]] (build 지시 누락)

## 관련 노트

- [[4. 지식노트/Pydantic - Python 데이터 검증 라이브러리.md]]
- [[4. 지식노트/Windows - 심볼릭 링크로 폴더 동기화.md]]
- [[3. 프로젝트/spx-agent/conventions.md]] § 빌드/도커
- [[3. 프로젝트/spx-agent/CLAUDE.md]] § 빠른 시작
