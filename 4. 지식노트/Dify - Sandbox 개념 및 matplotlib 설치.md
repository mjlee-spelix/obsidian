---
tags: [지식, dify, sandbox, docker]
date: 2026-03-25
---
# Dify - Sandbox 개념 및 matplotlib 설치

## 핵심
- Dify 코드 노드는 별도의 격리된 샌드박스(Docker 컨테이너)에서 실행됨
- Cloud 환경에서는 표준 라이브러리만 사용 가능 (`matplotlib`, `pandas` 등 불가)
- Self-hosted 환경에서는 Sandbox 컨테이너를 직접 수정해서 라이브러리 설치 가능

## 상세

### 샌드박스란?
외부와 격리된 안전한 실행 환경. 사용자 코드가 메인 시스템에 영향을 주지 않도록 격리함.

| 구분 | 설명 |
|---|---|
| 비유 | 실험실 안의 밀폐된 실험 박스 |
| 목적 | 위험한 코드가 메인 시스템을 망가뜨리지 않게 함 |
| Dify에서의 위치 | `dify-sandbox`라는 이름의 별도 Docker 컨테이너 |
| 해결책 | 라이브러리가 필요하면 샌드박스 컨테이너 자체에 설치해야 함 |

- **시스템 보호**: `os.remove('/')` 같은 위험 코드도 샌드박스 안에서만 실행됨
- **자원 제한**: 무한 루프 등으로 CPU/메모리 독점 방지
- **보안**: 악성 코드가 외부 네트워크로 나가는 것을 차단

### Self-hosted에서 matplotlib 설치하기

#### 방법 A. Docker Compose 환경 변수 수정
1. `docker-compose.yaml` 열기
2. `sandbox` 서비스의 `environment` 섹션 확인
3. 필요 시 커스텀 이미지 빌드

#### 방법 B. Sandbox Dockerfile 수정 (권장)
`dify/api/core/external_resource/sandbox/python/Dockerfile`에 아래 줄 추가:
```dockerfile
RUN pip install matplotlib numpy pandas
```
이후 재빌드:
```bash
docker-compose build sandbox
```

### 작업 순서 요약
1. Dify 로컬 설치 (Docker)
2. Sandbox 이미지에 라이브러리 추가 후 빌드
3. 코드 노드에서 Base64로 이미지 생성 후 String으로 반환
4. Answer 노드에서 마크다운으로 출력

## 관련 노트
- [[Dify - 코드 노드 시각화 구현]]
- [[Base64 인코딩]]
