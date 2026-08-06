---
tags: [지식, ngrok, dify, 네트워크, 트러블슈팅, 진단방법론]
date: 2026-05-27
---
# ngrok - 진단 1단계는 포워딩 대상 서비스 매핑 확인

## 핵심

- ngrok 관련 장애 발생 시 **첫 번째 확인 = ngrok이 어느 내부 서비스에 매핑되어 있는지 (포워딩 대상)**
- 매핑 확인 없이 포트 가설을 세우면 **오진** 위험 — 본인의 5/26 "5001 mismatch" 가설이 오진 사례
- 특히 Dify처럼 **다중 포트 구성** (Web 3000 / API 5001 / Gateway 등) 서비스는 매핑 확인이 필수
- ngrok URL이 살아있다고 해도 포워딩 대상이 죽어있거나 바뀌어 있으면 동일하게 `Connection refused`

---

## 1. 배경

Dify 기반 '사규 검색' 에이전트의 `Tool - RAG Chat HTTP 오류` (4월 21일부터 지속):

```
Reached maximum retries (0) for URL
https://hierogrammatic-supercongested-millie.ngrok-free.dev/v1/chat-messages
```

같은 Dify 인스턴스의 워크플로우(`RAG Chatbot`)를 **내부 호출 불가 이슈** 때문에 ngrok 경유 외부 HTTP로 우회 호출하는 특이 구조.

---

## 2. 오진 사례 — 본인이 박은 가설 (5/26)

> ngrok 80 포트 ↔ Dify API(dify-api) 5001 포트 mismatch → **Dify API를 5001로 재시작하면 해결**

이 가설은 **틀렸음** (5/27 정정):

- ngrok 매핑은 Dify API(5001)가 아니라 **Dify Web(3000)** 쪽과 연관
- 따라서 "5001로 재시작" 가설은 무효 — 재시작해도 ngrok 매핑이 안 잡힘
- 본인이 ngrok 포워딩 대상을 실측하지 않고 포트 번호만 보고 가설을 세운 게 원인

---

## 3. 올바른 진단 순서

### 3-1. ngrok 자체 살아있는지 (필요 조건)

```bash
ps aux | grep ngrok
# 또는
curl -s http://localhost:4040/api/tunnels | python3 -m json.tool
```

→ 본 케이스에서 ngrok은 3월 24일부터 정상 실행 중

### 3-2. 터널 URL 확인 (변경 여부)

```bash
curl -s http://localhost:4040/api/tunnels | grep public_url
```

→ 본 케이스에서 URL은 `hierogrammatic-supercongested-millie.ngrok-free.dev`로 동일 유지

### 3-3. **포워딩 대상 확인** ← 이 단계가 핵심

```bash
curl -s http://localhost:4040/api/tunnels | python3 -c "
import sys, json
data = json.load(sys.stdin)
for t in data['tunnels']:
    print(f\"{t['public_url']} → {t['config']['addr']}\")
"
```

본 케이스 결과: `https://...ngrok-free.dev → http://192.168.10.159:80`

### 3-4. 포워딩 대상이 살아있는지 직접 호출

```bash
curl -v http://192.168.10.159:80/health
# 또는 ngrok이 가리키는 정확한 path
curl -v http://192.168.10.159:80/v1/chat-messages
```

본 케이스 결과: `Connection refused` (포트 80이 죽어있음)

### 3-5. 포워딩 대상 서비스 ↔ 실제 서비스 매핑 확인 ← **여기서 오진 갈림**

- 80 포트가 누구 것인지? Dify Web(3000)? API(5001)? nginx gateway?
- 본 케이스: 80은 **Dify Web(3000)** 쪽 게이트웨이 — Dify API(5001)와 무관
- ❌ "5001 mismatch → API 재시작" 가설은 여기서 박살남

---

## 4. Dify 다중 포트 구성 (포워딩 대상 후보)

| 포트 | 서비스 | 역할 |
|------|--------|------|
| 3000 | dify-web (Next.js) | 사용자 UI / `/v1/chat-messages` 같은 API 프록시 |
| 5001 | dify-api (Flask) | 내부 API 본체 |
| 80 / 443 | nginx | 게이트웨이 (web/api 라우팅) |

ngrok이 80을 가리키면 nginx 경유 → 그 뒤가 Web인지 API인지는 **nginx 설정 봐야 알 수 있음**.
포트 번호만 보고 "5001 mismatch"라고 추론하면 안 됨.

---

## 5. 학습 포인트

- ngrok 진단 첫 단계는 **항상 포워딩 대상 매핑 확인** — URL이 살아있다는 사실은 매핑 정합을 보장하지 않음
- 다중 포트 서비스에서는 **포트 번호 ≠ 서비스 정체** — nginx/gateway 라우팅 한 단계 더 추적 필요
- ngrok URL이 "살아있는데 응답이 이상하다" 패턴이면:
  1. 포워딩 대상 살아있나? (`curl <internal-addr>`)
  2. 포워딩 대상이 ngrok 가리키는 path를 실제로 처리하나?
  3. 포워딩 대상 서비스가 **본인이 기대한 서비스 맞나?** (Web vs API 헷갈리기 쉬움)
- 본인이 박은 가설이 폐기되는 경험은 **다음 사건에서 같은 실수 안 하기 위한 박제** — 부끄러워하지 말 것

---

## 6. 본 케이스 미해결 상태

- 80 포트가 4월 21일 전후 왜 죽었는지 미확정
- ngrok-Dify Web 3000 매핑이 정확히 어떻게 nginx를 통해 가는지 추가 단서 필요
- **담당자 트랙으로 이관** (본인 추가 조사 종료)

---

## 관련 노트

- [[Docker - 컨테이너·이미지 기본 명령어]]
- [[vLLM - Responses API tool calling 버그 (gemma4 parser 미개입)]]
