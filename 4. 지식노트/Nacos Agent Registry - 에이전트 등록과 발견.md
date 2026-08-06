---
tags: [지식, AI-Agent, AI, 개발]
date: 2026-04-24
---
# Nacos Agent Registry - 에이전트 등록과 발견

## 핵심
- 알리바바의 오픈소스 **서비스 등록/발견 플랫폼 Nacos**를 AI 에이전트 영역으로 확장한 것
- 마이크로서비스의 "서비스 디스커버리"처럼 AI 에이전트를 등록하고 찾아주는 **에이전트 전화번호부**
- Nacos 3.1.0부터 A2A Agent Registry 지원

## 상세

### Nacos란?
- 알리바바가 2018년 오픈소스로 공개한 서비스 등록/발견 + 설정 관리 플랫폼
- 원래 마이크로서비스 환경에서 서비스 간 위치를 자동으로 찾아주는 역할
- 3.1.0 버전부터 이 개념을 AI 에이전트로 확장 → **A2A Agent Registry**

### 왜 필요한가?

```
❌ 하드코딩 방식 (Agent Registry 없이)
Master Agent:
  - 매출분석: http://192.168.1.10:8080  ← 주소 바뀌면 수정 필요
  - 전자결재: http://192.168.1.11:8080  ← 서버 죽으면 감지 불가
  - 새 Agent 추가 시 Master 수정 필요

✅ Agent Registry 방식
각 Agent → Nacos에 AgentCard 등록
Master → Nacos에 "매출 분석 가능한 Agent?" 조회 → 자동 발견
```

### 동작 원리 (3단계)

| 단계 | 설명 |
|-----|------|
| **① 등록** | Agent 시작 시 자신의 AgentCard(이름, 설명, 스킬, URL)를 Nacos에 등록 |
| **② 발견** | 다른 Agent가 Nacos에 "이런 작업 가능한 Agent 있어?"라고 조회 |
| **③ 호출** | 찾은 Agent의 URL로 A2A 프로토콜(JSON-RPC)을 통해 직접 호출 |

### AgentCard 관리 규칙

| 항목 | 내용 |
|-----|------|
| **고유성** | `namespace + name`으로 유일성 보장 |
| **이름 제약** | 최대 64자, ASCII 문자(32-126)만 허용 |
| **버전 관리** | 여러 버전 등록 가능, 기본 배포 버전(default published version) 지정 |
| **실시간 동기화** | 새 버전 배포 시 구독자에게 자동 알림 (listener 트리거) |

### 에이전트 등록 방식 3가지
1. **Spring AI Alibaba** - 자동 등록 및 통합 개발 프레임워크 활용
2. **SDK (Nacos-Client)** - Java 클라이언트를 통한 프로그래밍 방식
3. **HTTP API / Console** - 외부 에이전트의 수동 등록

### 에이전트 발견 모드 2가지
- **Nacos 모드** (권장): 자동 발견, 에이전트 이름만 지정하면 Nacos가 URL 반환
- **URL 모드**: 수동으로 각 Agent의 AgentCard URL을 직접 지정

### 마이크로서비스 vs Agent Registry 비교

| 비교 항목 | 마이크로서비스 (기존) | Agent Registry (AI) |
|---------|-----------------|-------------------|
| 등록 대상 | API 서비스 | AI 에이전트 |
| 메타데이터 | IP, 포트, 헬스체크 | AgentCard (스킬, 설명, 역량) |
| 발견 기준 | 서비스 이름 | 이름 + 스킬 + 태그 |
| 프로토콜 | HTTP / gRPC | JSON-RPC (A2A) |
| 헬스체크 | 하트비트 | 하트비트 + 버전 관리 |

### 지원 프로토콜
- JSONRPC, GRPC, HTTP+JSON 등 다중 전송 프로토콜 지원

## 참고 소스
- [Nacos A2A Registry 공식 문서](https://nacos.io/en/docs/latest/manual/user/ai/agent-registry/)
- [Nacos 공식 사이트](https://nacos.io/en/)
- [Nacos A2A Registry: AgentScope 크로스 프레임워크 연동 - Alibaba Cloud](https://www.alibabacloud.com/blog/nacos-a2a-registry-agentscope-enables-cross-language-and-cross-framework-interoperability_602821)
- [A2A Agent Registry 제안 - GitHub Discussion](https://github.com/a2aproject/A2A/discussions/741)

## 관련 노트
- [[A2A - Agent-to-Agent 프로토콜 개념]]
- [[Dify - A2A 플러그인 구현 방법]]
- [[Dify - 멀티 에이전트 설계 패턴]]
