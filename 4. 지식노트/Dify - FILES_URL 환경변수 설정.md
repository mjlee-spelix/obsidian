---
tags: [지식, dify, 개발]
date: 2026-03-26
---
# Dify - FILES_URL 환경변수 설정

## 핵심
- Self-hosted Dify에서 파일 첨부 시 상대 경로(`/files/...`)로 전달되어 이메일 등 외부 서비스에서 접근 불가
- `.env` 파일의 `FILES_URL`을 서버 주소로 설정하면 절대 경로로 변환됨

## 상세

### 에러 메시지
```
Failed to send email: Invalid file URL '/files/tools/3caad25e-....png':
Request URL is missing an 'http://' or 'https://' protocol.
Ensure the FILES_URL environment variable is set in your .env file
```

### 원인
도구/코드 노드에서 생성된 파일은 내부적으로 `/files/...` 상대 경로로 저장됨.
이메일 발송 등 외부 서비스는 `http://`로 시작하는 완전한 URL이 필요함.

### 해결 방법
Dify 설치 루트의 `.env` 파일 수정:

```bash
# 로컬 테스트
FILES_URL=http://localhost/files

# 외부 접속용 (IP 또는 도메인)
FILES_URL=http://<서버_IP_또는_도메인>:<포트>/files
# 예: FILES_URL=http://123.456.78.9:80/files
```

수정 후 재시작:
```bash
docker-compose down
docker-compose up -d
```

### 추가 확인 사항
- `CONSOLE_URL`, `APP_URL`도 동일한 도메인/IP로 맞춰야 함
- Nginx 등 리버스 프록시 사용 시 외부 접속 포트와 일치 여부 확인

## 관련 노트
- [[Dify - SMTP 이메일 발송 설정]]
- [[Dify - Sandbox 개념 및 matplotlib 설치]]
