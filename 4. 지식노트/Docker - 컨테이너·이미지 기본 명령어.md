---
tags: [개발, Docker]
date: 2026-04-23
---
# Docker - 컨테이너·이미지 기본 명령어

## 핵심
- Docker는 애플리케이션을 **컨테이너**라는 격리된 환경에서 실행하는 도구
- **이미지** = 설계도 (변경 불가), **컨테이너** = 이미지로 만든 실행 중인 인스턴스
- `docker ps`로 상태 확인, `docker logs`로 로그, `docker exec`로 컨테이너 내부 접속
- dify의 api, worker, worker_beat, nginx 등이 각각 하나의 컨테이너

## 상세

### 이미지 vs 컨테이너

```
이미지 (Image)                    컨테이너 (Container)
━━━━━━━━━━━━━━                   ━━━━━━━━━━━━━━━━━━
• 설계도/틀                       • 이미지로 만든 실행 환경
• 변경 불가 (읽기 전용)             • 읽기+쓰기 가능
• 한 이미지로 여러 컨테이너 생성     • 각 컨테이너는 독립적
• Docker Hub에서 pull              • start/stop/rm으로 관리
```

### 컨테이너 관리

#### docker ps — 컨테이너 목록

```bash
docker ps                      # 실행 중인 컨테이너만
docker ps -a                   # 모든 컨테이너 (중지된 것 포함)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"  # 보기 좋게 포맷
```

출력 읽는 법:
```
CONTAINER ID  IMAGE          STATUS         PORTS                  NAMES
a1b2c3d4e5f6  dify-api:1.0   Up 3 hours     0.0.0.0:5001->5001     dify-api-1
                              └ 실행 시간     └ 호스트:컨테이너 포트 매핑
```

#### docker start / stop / restart

```bash
docker start 컨테이너명           # 중지된 컨테이너 시작
docker stop 컨테이너명            # 정상 종료 (graceful)
docker restart 컨테이너명         # 재시작
docker stop $(docker ps -q)      # 실행 중인 전부 중지
```

#### docker logs — 로그 보기

```bash
docker logs 컨테이너명                    # 전체 로그
docker logs -f 컨테이너명                 # 실시간 로그 (tail -f처럼)
docker logs --tail 100 컨테이너명         # 최근 100줄만
docker logs -f --since "30m" 컨테이너명   # 최근 30분 + 실시간
docker logs 컨테이너명 2>&1 | grep ERROR  # 에러만 필터
```

- dify 로그 확인 예시: `docker logs -f dify-api-1`

#### docker exec — 컨테이너 내부 접속

```bash
docker exec -it 컨테이너명 bash          # bash 쉘로 접속
docker exec -it 컨테이너명 sh            # bash 없으면 sh로
docker exec 컨테이너명 ls /app           # 접속 없이 명령 실행
docker exec 컨테이너명 cat /app/logs/server.log  # 파일 내용 확인
```

- `-i` : interactive (입력 가능)
- `-t` : tty (터미널 할당)
- 나가기: `exit` 또는 `Ctrl+D`

#### docker rm — 컨테이너 삭제

```bash
docker rm 컨테이너명               # 중지된 컨테이너 삭제
docker rm -f 컨테이너명            # 실행 중이어도 강제 삭제
docker rm $(docker ps -aq)        # 모든 중지된 컨테이너 삭제
```

#### docker inspect — 상세 정보

```bash
docker inspect 컨테이너명                           # 전체 정보 (JSON)
docker inspect 컨테이너명 | grep IPAddress          # IP 주소만
docker inspect --format '{{.Config.Env}}' 컨테이너명 # 환경변수만
```

### 이미지 관리

```bash
docker images                      # 로컬 이미지 목록
docker images -a                   # 중간 이미지까지 전부

docker pull nginx:latest           # 이미지 다운로드 (태그 지정)
docker pull redis:7-alpine         # alpine = 경량 이미지

docker rmi 이미지명                 # 이미지 삭제
docker rmi $(docker images -q)     # 전체 삭제 (주의!)

docker tag 원본:태그 새이름:태그     # 이미지 태그 변경/복사
```

#### docker build — 이미지 빌드

```bash
docker build -t 이미지명:태그 .              # 현재 디렉토리의 Dockerfile로 빌드
docker build -t my-app:1.0 .               # 태그 지정
docker build -t my-app:1.0 -f Dockerfile.dev .  # Dockerfile 지정
```

### 기타 유용한 명령어

```bash
# 컨테이너 ↔ 호스트 파일 복사
docker cp 컨테이너명:/app/logs/server.log ./   # 컨테이너 → 호스트
docker cp ./config.yml 컨테이너명:/app/         # 호스트 → 컨테이너

# 포트 매핑 (컨테이너 실행 시)
docker run -p 8080:80 nginx     # 호스트 8080 → 컨테이너 80
docker run -p 127.0.0.1:8080:80 nginx  # localhost에서만 접근
```

포트 매핑 이해:
```
-p 호스트포트:컨테이너포트

예: -p 8081:8080
    브라우저에서 localhost:8081 접속
    → 컨테이너 내부의 8080 포트로 연결
```

### 자주 쓰는 실전 조합

```bash
# dify api 컨테이너에서 로그 확인
docker exec -it dify-api-1 tail -f /app/logs/server.log

# 컨테이너가 안 뜰 때 디버깅 순서
docker ps -a                          # 1. 상태 확인 (Exited?)
docker logs 컨테이너명                  # 2. 로그에서 에러 확인
docker inspect 컨테이너명               # 3. 설정/환경변수 확인

# 실행 중인 컨테이너 리소스 확인
docker stats                          # CPU/메모리 실시간 모니터링
docker stats --no-stream              # 현재 시점 스냅샷
```

## 관련 노트
- [[Docker - Docker Compose·실무 트러블슈팅]]
- [[Dify - Celery Beat 정기 백그라운드 작업]]
