---
title: spx-agent 에디션 · 기능 게이트 판별
phase: 번역 검증 하 P2 백로그 조사 (2026-06-16)
status: 확정 (이사님 "CE 버전이야" 확인, 2026-06-16)
audience: 기능 스코프 판정이 필요한 작성자·검수자
purpose: |
  spx-agent가 어떤 에디션인지, 그 결과 어떤 기능이 제품에 있고 없는지를
  코드 근거와 함께 박제. "유료/Enterprise 기능인가?"를 판정하는 방법을
  정전화해, i18n 문자열만 보고 오판하는 일을 막는다.
code_verified_at: 2026-06-16
---

# 1. 에디션 = CE (Community, self-hosted) 확정

- **이사님 확인 (2026-06-16): "CE 버전이야"**
- 코드 근거: `web/config/index.ts` L37 — `export const IS_CE_EDITION = EDITION === 'SELF_HOSTED'`
- spx-agent는 Dify 1.13.3 **커뮤니티 에디션 self-hosted 포크** → `EDITION='SELF_HOSTED'` → **`IS_CE_EDITION === true`**
- 함의: **유료(SaaS 구독)·Enterprise 라이선스 전용 기능은 제품에 없음** → 본문에서 다루지 않음(전역 규칙 #1과 동일 취급)

# 2. "유료/Enterprise 기능인가?" 판정 방법 (오판 방지)

> ⚠️ **i18n `upgradeFor*` 문자열만 보고 "플랜 게이트"라 단정하지 말 것.** 그 문자열은 클라우드 사용자에게 보일 CTA일 뿐, self-hosted 가용성과 직결되지 않는다. 본 조사 중 실제로 1차 오판함.

판정 순서:

1. **en 원문의 명시 문구 우선** — `"paid feature"`, `"SaaS subscription"`, `"Enterprise license"`, `dify.ai/pricing` 링크가 붙은 기능은 유료. CE 제외 강한 신호.
2. **컴포넌트 게이트 조건 확인** — `!IS_CE_EDITION` 조건으로 감싼 업그레이드 CTA는 **클라우드 전용 CTA**다. self-hosted에서는 그 CTA가 *숨겨질 뿐*, 기능이 자동으로 열리는 게 아니다(self-hosted 유료 경로 = Enterprise 라이선스).
3. **게이트가 아예 없으면 CE 보유** — `IS_CE_EDITION`·라이선스·`upgrade*` 체크가 전혀 없고 일반 hook으로 렌더되면 CE에서 사용 가능.

# 3. 확정 사례 (모델 제공자 자격 증명/로드 밸런싱)

| 기능 | 판정 | 코드/문서 근거 |
|------|------|--------------|
| **다중 자격 증명 관리** (모델당 여러 자격 증명 등록·전환, 기본 지정) | ✅ **CE 보유** | `web/app/components/header/account-setting/model-provider-page/model-auth/`(`manage-custom-model-credentials.tsx`, `credential-selector.tsx`)에 게이트 전무 — `IS_CE_EDITION`·라이선스·`upgrade*` 없음, `useCustomModels` 기반 렌더링만 |
| **로드 밸런싱** (여러 자격 증명 라운드로빈 분산) | ❌ **CE 미보유 (Enterprise/유료)** | en `workspace/model-providers.mdx` L161-163 명시: *"Load balancing is a paid feature — paid SaaS subscription **or Enterprise license**"* · `model-load-balancing-configs.tsx` L263 업그레이드 CTA가 `!modelLoadBalancingEnabled && !IS_CE_EDITION`(클라우드 전용) |

## 본문 반영 결과 (`ko/use-spx-agent/workspace/model-providers.mdx`)

- 다중 자격 증명 섹션 유지. 누락됐던 **"비용 최적화" 시나리오 복원** (환경 분리·비용 최적화·모델 테스트 3종 완비)
- **로드 밸런싱 섹션·"Default Config" `<Info>` 미작성 = 정상** (CE 미보유). en에 있어도 포팅 안 함

# 4. 같은 원리로 주의할 다른 후보

- AI 크레딧(`modelProvider.card.creditsExhausted`·`upgradePlan`) = 클라우드 빌링 → CE 무관, 제외
- 그 외 `upgradeFor*`·`dify.ai/pricing`·"Enterprise" 표기가 붙은 기능을 만나면 본 §2 절차로 판정 후 [[decisions]]에 사례 추가
