---
tags: [지식, AI-Agent, AI, dify, 개발]
date: 2026-04-24
---
# A2A - Agent-to-Agent 프로토콜 개념

## 핵심
- Google이 2025년 초 발표하고 Linux Foundation에 기부한 **AI 에이전트 간 통신 오픈 프로토콜**
- 서로 다른 프레임워크(Dify, LangChain, CrewAI 등)로 만든 에이전트끼리 **표준화된 방식으로 통신** 가능
- 기존 Tool-Use(도구 호출) 방식과 달리 에이전트가 **대등한 피어(peer)**로서 소통

## 상세

### Tool-Use 방식 vs A2A 방식 비교

| 구분 | Tool-Use (기존) | A2A |
|---|---|---|
| Agent 간 관계 | Master가 Sub를 **도구로 호출** | Agent가 **대등한 피어**로 통신 |
| 통신 프로토콜 | 플랫폼 내부 Function Calling | 표준화된 **JSON-RPC** |
| Agent 발견 | 하드코딩 (미리 등록) | **Agent Card**로 동적 발견 |
| 데이터 교환 | 노드 간 변수 전달 | **Task 객체 + Artifact** |
| 결합도 | 높음 (같은 플랫폼 필수) | 낮음 (프레임워크 무관) |

### A2A의 핵심 구성 요소

#### 1. Agent Card (에이전트 명함)
- `/.well-known/agent.json` 경로로 노출
- 에이전트의 이름, 설명, 스킬, URL, 버전 등 메타데이터 포함
- 다른 에이전트가 이 카드를 읽고 역량을 파악 후 호출 결정

```json
{
  "name": "sales-analysis-agent",
  "description": "매출 데이터 분석 및 차트 생성",
  "url": "https://domain.com/a2a",
  "version": "1.0.0",
  "skills": [
    {"name": "NL2SQL", "description": "자연어를 SQL로 변환"},
    {"name": "chart_generation", "description": "차트 이미지 생성"}
  ]
}
```

#### 2. Task (작업 단위)
- 에이전트 간 요청/응답의 기본 단위
- 생명주기: 생성 → 진행 중 → 완료/실패
- 동기(즉시 응답) / 비동기(장시간 작업) 모두 지원

#### 3. JSON-RPC
- 에이전트 간 실제 메시지 교환 프로토콜
- `message/send` 메서드로 에이전트에 메시지 전송
- 스트리밍(SSE) 응답 지원

### SPX Agent 데모 분석 사례
- SPX 데모는 A2A가 **아닌** Tool-Use 방식 (Master Agent가 Sub Agent를 도구로 호출)
- Dify/n8n 내부 Function Calling으로 통신
- A2A 전환 시 각 Sub Agent를 독립 서비스로 분리 필요

### A2A 도입의 장점
- 에이전트 **독립 배포/업데이트** 가능
- **다른 프레임워크**로 만든 Agent와도 호환
- Agent Card로 **동적 발견** (하드코딩 불필요)
- 느슨한 결합 → **확장성** 향상

## 참고 소스
- [A2A Protocol 공식 사이트](https://a2a-protocol.org/latest/)
- [A2A GitHub Repository](https://github.com/a2aproject/A2A)
- [MCP vs A2A: The Complete Guide to AI Agent Protocols in 2026 - DEV Community](https://dev.to/pockit_tools/mcp-vs-a2a-the-complete-guide-to-ai-agent-protocols-in-2026-30li)
- [A2A Protocol Explained: Secure Interoperability for Agentic AI 2026](https://onereach.ai/blog/what-is-a2a-agent-to-agent-protocol/)
- [Google Cloud Next 2026: AI agents, A2A protocol](https://thenextweb.com/news/google-cloud-next-ai-agents-agentic-era)

## 관련 노트
- [[Nacos Agent Registry - 에이전트 등록과 발견]]
- [[Dify - A2A 플러그인 구현 방법]]
- [[Dify - 멀티 에이전트 설계 패턴]]
