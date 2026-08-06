---
tags: [개발, Docker]
date: 2026-04-23
---
# Docker - Docker Compose·실무 트러블슈팅

## 핵심
- Docker Compose는 **여러 컨테이너를 한 번에** 관리하는 도구 — dify처럼 api, worker, db, redis 등이 같이 필요한 서비스에 필수
- `docker-compose.yml` 파일 하나로 전체 서비스 정의, `docker compose up -d`로 한 방에 시작
- 트러블슈팅 순서: `ps` → `logs` → `inspect` → `exec` 순서로 좁혀가기
- 디스크 부족은 `docker system prune`으로 해결

## 상세

### Docker Compose 기본 명령어

#### 서비스 시작/중지

```bash
docker compose up                   # 전체 서비스 시작 (포그라운드, 로그 보임)
docker compose up -d                # 백그라운드 실행 (detach) ← 보통 이걸 씀
docker compose up -d api worker     # 특정 서비스만 시작
docker compose up -d --build        # 이미지 다시 빌드하고 시작

docker compose down                 # 전체 중지 + 컨테이너 삭제
docker compose down -v              # 볼륨까지 삭제 ⚠️ DB 데이터 날아감!
docker compose down --rmi all       # 이미지까지 삭제 (완전 초기화)
```

> ⚠️ `docker compose down -v`는 볼륨(DB 데이터 등)도 삭제! 실서버에서 절대 주의

#### 상태 확인

```bash
docker compose ps                   # compose 서비스 상태
docker compose ps -a                # 중지된 것까지
docker compose top                  # 각 컨테이너 내부 프로세스
```

#### 로그

```bash
docker compose logs                 # 전체 서비스 로그
docker compose logs api             # api 서비스만
docker compose logs -f api          # api 실시간 로그
docker compose logs -f --tail 50 api worker  # 여러 서비스, 최근 50줄부터
```

#### 재시작/업데이트

```bash
docker compose restart              # 전체 재시작
docker compose restart api          # api만 재시작
docker compose pull                 # 이미지 최신 버전 다운로드
docker compose pull && docker compose up -d  # 업데이트 후 재시작

# 특정 서비스만 재빌드 + 재시작
docker compose up -d --build api
```

#### 컨테이너 접속

```bash
docker compose exec api bash        # api 서비스 컨테이너에 접속
docker compose exec api sh          # bash 없으면 sh로
docker compose exec db psql -U postgres  # DB 직접 접속
docker compose exec api cat /app/logs/server.log  # 파일 확인
```

### docker-compose.yml 구조 이해

```yaml
version: '3.8'

services:
  api:                               # 서비스 이름
    image: dify-api:latest           # 사용할 이미지
    build: ./api                     # 또는 Dockerfile 경로
    ports:
      - "5001:5001"                  # 포트 매핑 (호스트:컨테이너)
    environment:                     # 환경변수
      - DB_HOST=db
      - REDIS_HOST=redis
    env_file:
      - .env                         # .env 파일에서 환경변수 로드
    volumes:
      - ./app/logs:/app/logs         # 볼륨 마운트 (호스트:컨테이너)
    depends_on:                      # 의존성 (db가 먼저 시작)
      - db
      - redis
    restart: always                  # 항상 재시작

  db:
    image: postgres:15-alpine
    volumes:
      - db_data:/var/lib/postgresql/data  # named volume
    environment:
      - POSTGRES_PASSWORD=password

  redis:
    image: redis:7-alpine

volumes:
  db_data:                           # named volume 선언
```

핵심 필드:

| 필드 | 설명 | 예시 |
|------|------|------|
| `image` | 사용할 이미지 | `nginx:latest` |
| `build` | Dockerfile로 빌드 | `./api` |
| `ports` | 포트 매핑 | `"8080:80"` |
| `volumes` | 데이터 영속화 | `./data:/app/data` |
| `environment` | 환경변수 | `DB_HOST=db` |
| `env_file` | 환경변수 파일 | `.env` |
| `depends_on` | 시작 순서 | `db`, `redis` |
| `restart` | 재시작 정책 | `always`, `unless-stopped` |

### 볼륨과 네트워크

#### 볼륨 — 데이터 영속화

```bash
docker volume ls                    # 볼륨 목록
docker volume inspect 볼륨명         # 볼륨 상세 (실제 저장 경로)
docker volume rm 볼륨명              # 볼륨 삭제
docker volume prune                 # 안 쓰는 볼륨 전부 삭제
```

볼륨 타입:
```yaml
volumes:
  # 바인드 마운트: 호스트의 특정 경로 ↔ 컨테이너
  - ./my-data:/app/data

  # Named volume: Docker가 관리하는 볼륨
  - db_data:/var/lib/postgresql/data
```

#### 네트워크 — 컨테이너 간 통신

```bash
docker network ls                   # 네트워크 목록
docker network inspect 네트워크명    # 어떤 컨테이너가 연결돼 있는지
```

- 같은 compose 파일의 서비스들은 **자동으로 같은 네트워크**에 속함
- 서비스 이름으로 통신 가능 (예: api 컨테이너에서 `db:5432`로 접근)

### 환경변수 `.env` 파일 관리

```bash
# .env 파일 (docker-compose.yml과 같은 디렉토리)
DB_HOST=db
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=mypassword
REDIS_HOST=redis
SECRET_KEY=my-secret-key
```

```yaml
# docker-compose.yml에서 참조
services:
  api:
    environment:
      - DB_HOST=${DB_HOST}
    env_file:
      - .env          # 파일 통째로 로드
```

> ⚠️ `.env`는 `.gitignore`에 추가! 비밀번호가 Git에 올라가면 안 됨

### 트러블슈팅

#### 컨테이너가 안 뜰 때 디버깅 순서

```bash
# 1단계: 상태 확인
docker compose ps -a
# → Exited (1) = 에러로 종료, Restarting = 반복 재시작

# 2단계: 로그 확인
docker compose logs 서비스명
# → 에러 메시지 확인

# 3단계: 설정 확인
docker compose config
# → yml 문법 오류 검증

# 4단계: 컨테이너 내부 확인
docker compose exec 서비스명 bash
# → 파일/환경변수 직접 확인
```

#### 흔한 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 포트 충돌 | 호스트 포트 이미 사용 중 | `ss -tlnp \| grep 포트` → 변경 또는 기존 프로세스 종료 |
| 볼륨 권한 오류 | 컨테이너 내부 유저 권한 | `chmod`/`chown` 또는 Dockerfile에서 유저 설정 |
| 이미지 pull 실패 | 네트워크 또는 인증 | `docker login`, 프록시 설정 확인 |
| 디스크 부족 | 로그/이미지/볼륨 누적 | `docker system prune` |
| DB 연결 실패 | depends_on은 "준비"를 보장 안 함 | healthcheck 설정 또는 앱에서 retry |

#### 디스크 정리

```bash
docker system df                    # Docker가 사용하는 디스크 현황
docker system prune                 # 안 쓰는 컨테이너, 네트워크, 이미지 정리
docker system prune -a              # 사용 중이 아닌 이미지까지 전부 정리
docker system prune --volumes       # 볼륨까지 정리 ⚠️

# 개별 정리
docker image prune                  # 안 쓰는 이미지만
docker container prune              # 중지된 컨테이너만
docker volume prune                 # 안 쓰는 볼륨만
```

#### 리소스 모니터링

```bash
docker stats                        # 컨테이너별 CPU/메모리 실시간
docker stats --no-stream            # 현재 시점만 (스크립트용)
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### dify에서 자주 쓰는 명령어 모음

```bash
# 전체 시작/중지
docker compose up -d
docker compose down

# api 서비스 로그 실시간 확인
docker compose logs -f api

# worker 재시작 (코드 변경 반영)
docker compose restart worker

# api 컨테이너 안에서 로그 확인
docker compose exec api tail -f /app/logs/server.log

# DB 직접 접속
docker compose exec db psql -U postgres

# 이미지 업데이트 후 재시작
docker compose pull && docker compose up -d
```

## 관련 노트
- [[Docker - 컨테이너·이미지 기본 명령어]]
- [[Dify - Celery Beat 정기 백그라운드 작업]]
- [[Dify - 토큰 데이터 저장 흐름 (동기·비동기)]]
