# 외부 연결 처리 framing 제안 (이사님 컨펌용) — **종결 (2026-06-04)**

> ⛔ **본 문서는 컨펌 자료로 작성됐고 2026-06-04 회의로 가결과 적용 완료.**
> 이후 활용 없음. 참고용 보존.
>
> **결과**: 이사님 회의에서 옵션 A(deployer 결정 영역) 채택 X. **외부 연결(outbound) 명시 제거** 결정. 자세히는 [[../decisions#2026-06-04 — 이사님 회의 결정 일괄 박제]] §결정 2 참조.
>
> **반영 위치**:
> - [[../scope-mapping]] — 매트릭스 14건 행 ❌ 삭제 적용 (Monitor 7 / 지식 외부 import 3 / 외부 KB 2 / Twitter / API Extension 3)
> - [[../scope-mapping#전역 적용 규칙]] §6 신설 (KC 추상화)
> - inbound 3건은 **유지·번역** 확정 ([[../decisions]] §결정 4)
>
> 작성: 2026-06-04
> 목적 (당시): spx-agent docs에서 "외부 연결" 관련 페이지 22건의 처리 방향 일괄 결정

## 1. 문제 정의

Phase 1.3 매트릭스 검토 중 ⚠️ **검토(보류)** 상태인 페이지가 **22건** 남아있음. 모두 "외부 서비스와의 연결" 관련:

- spx-agent → 외부 서비스 호출 (outbound)
- 외부 → spx-agent 호출 (inbound)
- 특정 기능 활성화 (Marketplace/MCP/Plugin Trigger)

페이지별로 사내 정책 일일이 확인하려면 시간 소요 큼 + 매번 재판단 필요. **framing 한 번 정해두면 일괄 적용 가능**.

## 2. 제안 framing

**spx-agent = 솔루션 제품**으로 포지셔닝 → "외부 연결 가능 여부"는 **두 가지 의사결정 층위**로 분리:

| 층위 | 결정 주체 | 예시 |
|------|---------|------|
| **Product 수준** (제품 자체 활성화/비활성화) | Spelix (제조사) | Marketplace 활성화, MCP 활성화, Plugin 시스템 활성화 |
| **Deployment 수준** (배포 환경별 정책) | 각 deployer (배포 고객사) | 방화벽 outbound 정책, API 외부 노출, 외부 KB 연결 |

## 3. 페이지 22건 분류

### Product 수준 — Spelix 결정 (3건)

| 페이지 | 항목 | 현재 |
|--------|------|------|
| nodes/trigger/plugin-trigger | Plugin Trigger (langbot/lark/telegram 등 Marketplace 플러그인) | 검토 |
| build/mcp | MCP 사용 안내 | 검토 |
| publish/publish-mcp | MCP 게시 | 검토 |

→ Marketplace 결정 시 Plugin Trigger 자동 처리 (Marketplace 없으면 langbot 등 사용 불가)
→ MCP는 Marketplace와 독립 (별도 결정 필요)

### Deployment 수준 — Deployer 결정 (19건)

| 카테고리 | 페이지 | 본질 |
|---------|--------|------|
| 외부 SaaS observability 송출 | monitor/integrations/* (LangSmith·Langfuse·Opik·Weave·Arize·Phoenix·Aliyun) — 7건 | spx-agent → 외부 trace 송출 |
| 외부 API 호출 (튜토리얼) | tutorials/twitter-chatflow | Twitter API |
| 외부 사이트 노출 | publish/webapp/embedding-in-websites | 외부 사이트에 spx-agent WebApp iframe |
| 외부 데이터 import (지식 a) | sync-from-notion, sync-from-website, authorize-data-source — 3건 | 외부 데이터 → spx-agent KB |
| 외부 KB read-only 연결 (지식 b) | connect-external-knowledge-base, external-knowledge-api — 2건 | 외부 KB ← spx-agent |
| 외부 노출 API (c) | developing-with-apis, maintain-dataset-via-api, workspace/api-extension/* — 5건 | 외부 시스템이 spx-agent를 API로 호출 |

## 4. 권장 처리

### 옵션 A — 제안 (적극)

**Deployment 수준 19건 = 유지·번역 (기능 설명만, 정책은 deployer 결정 영역)**

| 옵션 | 처리 |
|------|------|
| 매트릭스 액션 | ⚠️ 검토 → ✅ 유지·번역 (19건) |
| 본문 메모 | 없음 — 매뉴얼은 기능을 설명, 정책은 deployer 가이드 따로 |
| 검토 묶음 표 | 6분류 → 1분류 (Marketplace + MCP만) |
| 컨펌 부담 | 1건 (Marketplace) + 1건 (MCP)로 축소 |

장점: Phase 4 진입 속도 ↑, 페이지별 정책 판단 부담 0, 사용자 매뉴얼이 deployment 메모로 도배되지 않음
단점: 일부 기능을 사용자가 "써봤더니 안 되는" 케이스 발생 가능 — 그러나 UI에서 즉시 확인 가능한 영역이라 docs로 막을 이유 약함

### 옵션 B — 보수 (현 상태 유지)

19건도 페이지별로 사내 정책 확인 후 결정. 작업량 크고 결정 미루기 누적.

### 옵션 C — 중간

19건 유지·번역하되 "이 기능은 deployment 환경에 따라 사용 가능 여부가 다를 수 있습니다" 메모 한 줄 추가.

장점: deployer가 명시적 안내
단점: 매뉴얼 노이즈 ↑, 한국어 자연스러움 ↓

## 5. 컨펌 항목 (요약)

### 핵심 — framing 채택 여부

- [ ] **(가) 옵션 A 채택** (Deployment 수준은 deployer 결정 영역, 페이지 살림) — 권장
- [ ] **(나) 옵션 B 채택** (기존대로 페이지별 정책 확인)
- [ ] **(다) 옵션 C 채택** (살리되 deployment 메모 추가)

### 부차 — Product 수준 결정

- [ ] **Marketplace 활성화 여부**
  - (가) 비활성화 — Marketplace 페이지 삭제, Plugin Trigger 자동 비활성, 기능 확장 안내 폐기
  - (나) 활성화 — 페이지 유지·번역
- [ ] **MCP 활성화 여부**
  - (가) 활성화 — 페이지 유지·번역 (권장: Marketplace와 독립, 외부 MCP 서버 호출 가능)
  - (나) 비활성화 — 페이지 삭제

## 6. 컨펌 후 후속 작업

옵션 A + Marketplace 비활성 + MCP 활성 채택 시:

| 작업 | 영향 |
|------|------|
| 매트릭스 22건 액션 일괄 변경 | 검토 22 → 검토 0, 유지·번역 +19, 삭제 +1~3 (Marketplace 관련 / Plugin Trigger) |
| 검토 대기 표 6분류 → 0~1 | Marketplace 컨펌 결과 따라 폐기 또는 단일 항목 |
| 전역 규칙 #5 단순화 | Marketplace 항목·기능 확장 추출 항목 단순화/제거 |
| feature-extension-extracts.md 폐기 | Marketplace 결정과 묶임 |
| 챕터별 사전 분석 표 정리 | "컨펌 후 적용" 7개 → 1개 (Marketplace) |
| Phase 4 즉시 작업 가능 페이지 확장 | ~64p → ~84p |
| 결정 박제 | [[../decisions]] "2026-06-04 — Product vs Deployment framing 채택" |

## 7. 부록 — 분류 근거 상세

### Plugin Trigger 동작 확인

UI에서 트리거 노드 추가 시 표시되는 목록:
- 추천: langbot_trigger, lark_trigger, telegram_trigger, outlook_trigger, gmail_trigger (Marketplace 플러그인)
- 모든 트리거: 일정 트리거, 웹훅 트리거 (빌트인)

→ Marketplace 비활성 시 추천 목록만 비고 빌트인은 살아남음 → Plugin Trigger 페이지는 Marketplace 결정 따름

### Deployment 수준 분류 근거

- 외부 서비스 호출 가능 여부 = **방화벽 설정** (deployer 인프라 차원)
- API 외부 노출 = **endpoint 노출 정책** (deployer 보안 차원)
- 외부 KB 연결 = **사내 인프라 보유 여부** (deployer 자원 차원)

→ 모두 spx-agent 제품 자체 활성화/비활성화 결정 영역 X, deployer 환경 의존
