# Dify - Sandbox 나눔고딕 한글 폰트 설치

## 환경
- 서버: `/opt/dify/docker`
- Sandbox 이미지: `svcvit/dify-sandbox-py:0.1.4`
- OS: Debian (bookworm)

## 배경
matplotlib으로 차트를 그릴 때 한글이 깨지는 문제 해결.  
Dify sandbox는 Linux 환경이므로 나눔고딕 설치 필요.

## 설치 과정

### 1. Dockerfile.sandbox 생성
`/opt/dify/docker` 경로에서 실행:

```bash
cat > Dockerfile.sandbox << 'EOF'
FROM svcvit/dify-sandbox-py:0.1.4

USER root
RUN apt-get update && apt-get install -y fonts-nanum fontconfig && \
    fc-cache -fv && \
    find / -name "fontlist*.json" -delete 2>/dev/null
EOF
```

- `fonts-nanum`: 나눔 폰트 패키지
- `fontconfig`: `fc-cache` 명령어 포함, 폰트를 시스템에 등록하는 도구

### 2. docker-compose.yaml 수정
sandbox 섹션에서 `image:` 줄 삭제 후 `build:` 추가:

```yaml
sandbox:
  build:
    context: .
    dockerfile: Dockerfile.sandbox
  restart: always
  ...
```

### 3. 빌드 및 적용

```bash
docker-compose up -d --build sandbox
```

### 4. 설치 확인

```bash
docker exec docker-sandbox-1 fc-list | grep -i nanum
```

## 설치된 폰트 목록

| 폰트명 | 스타일 |
|--------|--------|
| NanumGothic (나눔고딕) | Regular, Bold |
| NanumBarunGothic (나눔바른고딕) | Regular, Bold |
| NanumGothicCoding (나눔고딕코딩) | Regular, Bold |
| NanumMyeongjo (나눔명조) | Regular, Bold |
| NanumSquare (나눔스퀘어) | Regular, Bold |
| NanumSquareRound (나눔스퀘어라운드) | Regular, Bold |

## 트러블슈팅

### fc-cache not found 오류
첫 빌드 시 `fontconfig` 없이 `fonts-nanum`만 설치해서 실패.  
→ `fontconfig` 패키지를 함께 설치해서 해결.

### 빌드 후 unhealthy 상태
sandbox 기동 시 matplotlib 등 의존성 패키지를 자동 설치하므로 3~4분 소요.  
로그에서 `Application startup complete` 확인 후 정상 사용 가능.

```bash
docker logs docker-sandbox-1 --tail 50
```

## 참고
- 워크플로우 데이터는 postgres에 저장되므로 sandbox 재빌드 시 데이터 유지
- 기존 설정 복구 필요 시: `cp docker-compose.yaml.org docker-compose.yaml`
- matplotlib 한글 폰트 우선순위 (Linux): `NanumGothic` → `NanumBarunGothic` → `UnDotum`
