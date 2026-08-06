# 검수 하네스 (지속 보관)

> 자동 1차 검수를 돌리는 도구. **규칙은 [[_rules-brief]] 한 곳에서만 관리**, 스크립트는 그룹 설정만 받는다.

## 파일 구성
| 파일 | 역할 |
|------|------|
| [[_rules-brief]] | **규칙 단일 출처** — 검수 에이전트가 읽고 적용. 전역규칙·확정삭제·유지·보강·conventions(문체·존댓말·영문병기·직역체·데이터시각화) + 권위 문서 경로. **규칙 수정은 여기만** |
| `review-harness.js` | 범용 워크플로 스크립트. `args`로 그룹 설정만 받음. 규칙은 브리프를 읽어 적용 |
| README (본 파일) | 실행법 + 그룹별 설정(복붙용) |

## 실행법

```
Workflow({
  scriptPath: "C:\\Users\\Administrator\\Projects\\spx-agent-docs\\.claude\\docs\\review\\harness\\review-harness.js",
  args: { …그룹 설정… }
})
```

- 백그라운드 실행 → 완료 시 결과(JSON) 알림. 결과를 해당 `0N-그룹.md`의 4영역 + 페이지별에 정리.
- 규칙을 바꾸려면 `review-harness.js`가 아니라 **[[_rules-brief]]만** 수정 (에이전트가 매 실행 시 그 파일을 읽음).

## args 스키마
```
{
  group:   "게시",                                  // 표시용 그룹명
  koBase:  "...\\ko\\use-spx-agent\\publish",       // ko 그룹 폴더 절대경로
  enBase:  "...\\en\\use-dify\\publish",            // en 원본 그룹 폴더
  i18nNote:"...\\ko-KR\\ 의 share.json, app.json",  // 이 그룹 i18n 파일 안내 문자열
  pages: [
    { rel: "readme.mdx",
      action: "부분 수정",                           // 유지·번역 | 부분 수정 | 신규 챕터
      intended: "권한 보강은 의도된 것",             // (선택) 원문에 없어도 정상인 추가
      note: "inbound 유지(결정4)",                   // (선택) 헷갈리기 쉬운 참고
      newChapter: true,                             // (선택) 원문 없음 → 원문대조 생략
      analysis: ["spx-knowledge-permissions.md"] }  // (선택) 분석본 정합 대조 대상
  ]
}
```
> 경로 base는 보통 `C:\Users\Administrator\Projects\spx-agent-docs\...` (ko/en) + `...\spx-agent\web\i18n\ko-KR` (i18n).

## 새 그룹 추가
1. `ko/use-spx-agent/<그룹>/` 페이지 목록 확인
2. [[scope-mapping]]에서 그룹 행의 **액션**(유지/부분수정/삭제)·삭제 페이지·부분수정 보강 확인 → `pages[].action`/`intended`/`note`에 반영
3. [[chapter-writing-checklist#4]]에서 그룹 i18n 파일 확인 → `i18nNote`
4. 신규 챕터면 분석본(`references/spx-*.md`)을 `analysis`에 추가
5. 위 args로 실행

## 완료 그룹 설정 (복붙용)

### 게시 (완료 2026-06-12)
```json
{ "group": "게시",
  "koBase": "C:\\Users\\Administrator\\Projects\\spx-agent-docs\\ko\\use-spx-agent\\publish",
  "enBase": "C:\\Users\\Administrator\\Projects\\spx-agent-docs\\en\\use-dify\\publish",
  "i18nNote": "...\\web\\i18n\\ko-KR 의 share.json, app.json, app-api.json",
  "pages": [
    { "rel": "readme.mdx", "action": "부분 수정", "intended": "Marketplace 게시 제거(결정1)+권한 챕터 링크. README의 Marketplace·web-app-access 빠진 건 정상" },
    { "rel": "webapp/workflow-webapp.mdx", "action": "유지·번역" },
    { "rel": "webapp/chatflow-webapp.mdx", "action": "유지·번역" },
    { "rel": "webapp/web-app-settings.mdx", "action": "유지·번역" },
    { "rel": "webapp/embedding-in-websites.mdx", "action": "유지·번역", "note": "inbound 유지(결정4)" },
    { "rel": "publish-mcp.mdx", "action": "유지·번역", "note": "MCP 유지(결정3)" },
    { "rel": "developing-with-apis.mdx", "action": "유지·번역", "note": "inbound 유지(결정4)" }
  ] }
```

### 지식 (완료 2026-06-12 — 구버전 하네스로 실행, 수동 보정 완료)
- 18p. i18n: dataset*.json, pipeline.json, dataset-pipeline.json. 부분수정 3p(권한 보강), maintain-dataset-via-api inbound, permissions 신규(분석본 spx-knowledge-permissions·spx-app-permissions-analysis).

## 주의
- 이전 세션 작업 폴더(`~/.claude/projects/<세션>/workflows/scripts/`)에 있던 `review-knowledge-group-*.js`·`review-publish-group-*.js`는 **이 범용 하네스로 대체**됨. 그 파일들은 세션 종료 시 휘발 가능 — 정전은 본 폴더.
