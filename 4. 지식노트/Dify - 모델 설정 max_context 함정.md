---
tags: [지식, dify, LLM, 모델설정, 트러블슈팅]
date: 2026-05-26
---
# Dify - 모델 설정 max_context 함정

## 핵심

- Dify는 모델 등록 시 UI에서 **`max_context_length`(또는 Context Size)를 강제로 잡음** — 이 값이 실제 모델 한계보다 작으면 입력 길이가 그 값을 넘는 순간 에러
- **모델이 128K 지원해도 Dify UI에서 `4096`으로 잡혀있으면 4096이 한계가 됨** — 모델 한계가 아니라 Dify 설정이 한계
- 운영 지침: **모델 실제 한계까지 풀어두는 게 안전** (`max_context_length`는 "절대 넘기지 못할 한도" 의미가 아니라 Dify가 잘라낼 임계점)

---

## 1. 증상

- 데모 워크플로우에서 `gpt-oss-120b → gemma-4-26b` 교체 후 일부 요청에서 에러
- 에러 메시지: 입력 길이 초과 또는 컨텍스트 한도 관련 에러
- 그러나 **모델 자체는 128K 컨텍스트 지원** (gemma-4-26b)

---

## 2. 원인

Dify 모델 등록 시 UI 폼:

```
Models → Add Model → gemma-4-26b
├── Context Size: 4096      ← 여기 ⚠️
├── Max Tokens: 2048
└── ...
```

`Context Size` 필드를 작게 잡으면:

- Dify가 LLM에 요청 보내기 전에 **입력 길이를 이 값으로 자름**
- 또는 입력이 이 값을 넘으면 **요청 자체를 거부**
- 모델이 실제로 더 받을 수 있어도 **Dify 단에서 차단**

---

## 3. 해결

`Models → 해당 모델 편집 → Context Size`를 모델 실제 한계로 늘림

| 모델 | 실제 한계 | Dify Context Size 권장 |
|------|-----------|-------------------------|
| gemma-4-26b | 128K | `131072` 또는 `128000` |
| gpt-oss-120b | 128K | `131072` |
| bge-m3-ko (embedding) | 8K | `8192` (vLLM `--max-model-len`과 일치) |

→ 모델 한계에 맞춰 풀어두는 게 안전

---

## 4. 왜 함정인가

- Dify UI 폼이 `4096` 같은 기본값을 박는 경우가 있음 (플러그인/제공자에 따라)
- 모델 교체 시 **이 값을 그대로 두면 새 모델의 큰 컨텍스트를 못 살림**
- 에러 메시지가 "Dify 설정 한계 초과"보다 "입력 너무 김" 같은 일반 메시지로 나와서 **모델 한계인 줄 오해**하기 쉬움
- 모델별로 한계가 달라서, **`Context Size`는 모델 교체 체크리스트에 반드시 포함**할 항목

---

## 5. 모델 교체 체크리스트 (관련)

Dify에서 모델을 다른 모델로 교체할 때 함께 확인:

- [ ] `Context Size` 새 모델 한계로 조정
- [ ] `Max Tokens` (출력 한도) 새 모델 한계로 조정
- [ ] vLLM `--tool-call-parser` / `--reasoning-parser` 새 모델에 맞게 교체 (gpt-oss → gemma4 등)
- [ ] tool calling 동작 확인 — 모델별 tool 포맷 차이 있음 (특히 비-OpenAI 모델)
- [ ] 워크플로우 종단 테스트 — 응답 포맷 변경 영향 확인

→ [[vLLM - Responses API tool calling 버그 (gemma4 parser 미개입)]]

---

## 6. 학습 포인트

- 추상화 계층(Dify) 위에서 모델을 다룰 때, **추상화 계층의 설정이 모델 한계보다 작을 수 있음** — 추상화 설정이 진짜 한계가 됨
- 에러 메시지가 일반적일수록 **상위 계층 설정도 의심 후보** — 모델/플러그인/Dify/네트워크 모두 한 번씩
- 모델 교체 = 단순 모델명 변경이 아니라 **컨텍스트 크기 / 출력 한도 / parser / tool 포맷** 4종 같이 점검

---

## 관련 노트

- [[Dify Agent - gemma-4 vLLM Tool Calling 트러블슈팅]]
- [[vLLM - Responses API tool calling 버그 (gemma4 parser 미개입)]]
- [[Dify - vLLM 임베딩 모델을 OpenAI-API-compatible로 등록]]
- [[Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름]]
