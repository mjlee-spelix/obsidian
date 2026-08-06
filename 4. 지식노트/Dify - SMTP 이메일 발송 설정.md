---
tags: [지식, dify, 개발]
date: 2026-03-26
---
# Dify - SMTP 이메일 발송 설정

## 핵심
- Dify 이메일 노드 사용 시 SMTP 설정 필요
- 일반 로그인 비밀번호가 아닌 **앱 비밀번호** 사용해야 함 (Gmail, Naver 모두 해당)

## 상세

### 주요 서비스별 설정값

| 항목                   | Gmail                      | Naver              |
| -------------------- | -------------------------- | ------------------ |
| **smtp server**      | `smtp.gmail.com`           | `smtp.naver.com`   |
| **smtp server port** | `465` (SSL) 또는 `587` (TLS) | `465`              |
| **encrypt method**   | `SSL` 또는 `STARTTLS`        | `SSL`              |
| **email account**    | 전체 이메일 주소                  | 전체 이메일 주소          |
| **email password**   | **앱 비밀번호 (16자리)**          | **앱 비밀번호** 또는 비밀번호 |

### 앱 비밀번호 발급 방법

#### Gmail
1. 구글 계정 설정 > 보안 > **2단계 인증** 활성화
2. 2단계 인증 메뉴 하단 > **앱 비밀번호** 클릭
3. 이름(예: Dify) 입력 → 생성된 **16자리 코드** 복사
4. Dify `email password` 칸에 입력

#### Naver
1. 네이버 메일 > 환경설정 > **POP3/IMAP 설정**
2. **POP3/SMTP 사용** → '사용함' 체크 후 저장
3. 2단계 인증 사용 중이면 앱 비밀번호 별도 생성 필요

## 관련 노트
- [[Dify - FILES_URL 환경변수 설정]]
