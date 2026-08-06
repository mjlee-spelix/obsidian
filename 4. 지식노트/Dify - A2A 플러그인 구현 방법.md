---
tags: [지식, dify, AI-Agent, AI, 개발]
date: 2026-04-24
---
# Dify - A2A 플러그인 구현 방법

## 핵심
- Dify는 **A2A Server** + **A2A Client(Discovery)** 두 플러그인으로 양방향 A2A 통신 지원
- A2A Server: 내 Dify 앱을 외부에 A2A 에이전트로 **노출**
- A2A Client: 외부 A2A 에이전트를 Dify에서 **호출**

## 상세

### 아키텍처 개요

```
┌────────────────────────────────────┐
│       Nacos Agent Registry         │
│     (에이전트 자동 발견/등록)         │
└───────┬────────────────┬───────────┘
        │ 등록             │ 발견
        ▼                 ▼
┌──────────────┐   ┌──────────────┐
│  Dify App A   │   │  Dify App B   │
│ (A2A Server)  │◄─►│ (A2A Client)  │
│ AgentCard 노출 │   │ 외부 Agent 호출│
└──────────────┘   └──────────────┘
```

### 1. A2A Server 플러그인 (내 Agent를 외부에 노출)

#### 설치
1. Dify Marketplace 또는 GitHub Releases에서 다운로드
2. Dify → Plugins → 패키지 업로드 → 설치

#### 엔드포인트 설정

| 파라미터 | 설명 | 예시 |
|---------|------|------|
| Dify App | 노출할 앱 선택 | `매출 분석 Agent` |
| Agent Name | 에이전트 이름 | `sales-analysis-agent` |
| Agent Description | 기능 설명 | `"매출 데이터 분석 및 차트 생성"` |
| Agent Public URL | 공개 접근 URL | `https://domain.com/e/{endpoint_id}/a2a` |
| Agent Version | 버전 | `1.0.0` |

> ⚠️ endpoint_id는 저장 후 Dify가 자동 생성하므로, 저장 후 URL 수정 필요

#### 생성되는 엔드포인트
- **AgentCard**: `GET /.well-known/agent.json` (발견용)
- **JSON-RPC**: `POST /a2a` (호출용)

#### 지원 앱 타입
- Chatbot, Agent, Chatflow, Workflow 모두 가능

#### Nacos 자동 등록
- Enable Nacos Registration 활성화 시, 첫 AgentCard 요청에서 자동 등록

### 2. A2A Client 플러그인 (외부 Agent 호출)

#### 설치
1. Dify Marketplace 또는 GitHub에서 `dify-a2a-plugin.difypkg` 다운로드
2. 자체 호스팅 시 환경변수 추가: `PLUGIN_ENABLE_SIGNATURE_VERIFICATION=false`
3. Dify → Plugins → Install via Local File → 설치

> ⚠️ Cloud Edition은 아직 미지원, 자체 호스팅 필요

#### 에이전트 등록 (최대 5개)

| 파라미터 | 설명 | 예시 |
|---------|------|------|
| Name | 에이전트 식별자 | `sales_agent` |
| Base URL | A2A 엔드포인트 | `https://agent.example.com` |
| Auth Type | 인증 방식 | None / Bearer Token / API Key / Basic Auth |
| Description | 설명 | `"매출 분석 전문 에이전트"` |

#### 제공 도구 5가지

| 도구 | 용도 | 비고 |
|-----|------|------|
| List Agents | 등록된 에이전트 목록 조회 | 초기 확인용 |
| Get Agent Capabilities | AgentCard 조회 (스킬/역량) | 첫 사용 전 확인 |
| Call Agent (Sync) | 동기 호출 | 60초 타임아웃, 빠른 응답용 |
| Submit Task (Async) | 비동기 작업 제출 | SSE 스트림, 장시간 작업용 |
| Get Task Status | 비동기 작업 상태 확인 | taskId 기반 폴링 |

### 3. A2A Discovery 플러그인 (Nacos 연동 발견)

#### 발견 모드 2가지

**Nacos 모드 (권장):**
```
discovery_type: nacos
nacos_address: mse-xxx.nacos.mse.aliyuncs.com:8848
available_agent_names: sales_agent,etl_agent,hr_agent
```

**URL 모드 (수동):**
```
discovery_type: url
available_agent_urls: {
  "sales_agent": "http://host1:8080/.well-known/agent.json"
}
```

### 주의사항

| 항목 | 내용 |
|-----|------|
| Dify 버전 | 플러그인 지원 버전 필요 |
| Python | 3.12+ |
| 프로토콜 | A2A v0.3.0 호환 필요 |
| 동기 호출 제한 | 60초 → 오래 걸리면 비동기(Submit Task) 사용 |
| Cloud Edition | A2A Client 미지원 (자체 호스팅만) |

### 기존 Tool-Use → A2A 전환 예시 (SPX 데모 기준)
```
기존: Master Agent → [도구] Sub Agent (같은 플랫폼 내부)
전환: Master Agent → [A2A Call] 독립 Agent (독립 서비스)
```
각 Sub Agent를 독립 Dify 앱으로 분리 → A2A Server로 노출 → Master에서 A2A Client로 호출

## 참고 소스
- [A2A Server - Dify Marketplace](https://marketplace.dify.ai/plugin/nacos/a2a_server?language=en)
- [A2A Client Plugin for Dify - Dify Marketplace](https://marketplace.dify.ai/plugin/ryan_duff/dify-a2a-plugin)
- [Dify Nacos A2A Plugin 공식 발표 - Alibaba Cloud](https://www.alibabacloud.com/blog/dify-officially-launched-the-nacos-a2a-plugin-completing-its-bidirectional-multi-agent-collaboration-capabilities_602852)
- [Support for the A2A protocol - Dify GitHub Issue](https://github.com/langgenius/dify/issues/19352)

## 관련 노트
- [[A2A - Agent-to-Agent 프로토콜 개념]]
- [[Nacos Agent Registry - 에이전트 등록과 발견]]
- [[Dify - 멀티 에이전트 설계 패턴]]
- [[Dify - 워크플로우 도구 버전 동기화]]
