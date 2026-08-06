# spx-agent-docs

Dify 공식 docs(영어판)를 fork해서 spx-agent용 한국어 매뉴얼/교육 자료로 재구성하는 프로젝트.

## 작업 위치
- **코드/MDX**: `C:\Users\Administrator\Projects\spx-agent-docs\` (원본 fork — Phase 3.5에서 Mintlify → Docusaurus 빌드 전환 예정, [[docs/decisions]] 2026-06-02)
- **진행 메모**: `C:\Users\Administrator\Documents\Obsidian\Daily\3. 프로젝트\spx-agent-docs` (코드 폴더의 `.claude/`에 junction 연결)

## 출처 / 라이선스
- 원본: https://github.com/langgenius/dify-docs
- 라이선스: **CC BY 4.0** (수정·배포·상업적 사용 가능, 출처 표기 의무)
- 의무 처리: 사내 fork의 `NOTICE.md`에 원본 명시 + LICENSE 파일 유지

## 원본 보존 원칙 (최우선)

**원본 파일은 최대한 손상 없이 유지**합니다. 우리 변경은 다음 형태로 표현:

- **추가 우선**: `ko/` 신설, 신규 챕터, `NOTICE.md` 같은 신규 파일
- **외부 표현**: 원본 규칙을 덮어쓰는 내용은 `.claude/CLAUDE.md` 등 별도 위치에서 표현 (루트 `CLAUDE.md` 미수정)
- **불가피한 수정 시 최소화**: `docs.json`(한국어 등록·ja/zh nav 비활성), `.gitignore`(옵시디언 분리) 등 부득이한 경우만, 라인 단위 변경

상세 방침과 영역별 처리는 `.claude/CLAUDE.md` "원본 보존 원칙" 섹션 참조.

## 폴더 구조

```
spx-agent-docs/        ← 옵시디언 (코드 폴더의 .claude로 junction)
├── CLAUDE.md          ← spx fork overrides (세션 시작 시 명시 로드 권장)
├── README.md          ← 본 파일 (vault 진입점 지도)
├── docs/              ← 프로젝트 메모
│   ├── scope-mapping.md
│   ├── conventions.md
│   ├── progress.md
│   ├── decisions.md
│   └── references/
│       ├── dify-docs-structure.md
│       └── ibm-research.md
└── skills/            ← Dify 원본 Claude Code 작성 보조 스킬 (보존)
```

- **루트 `CLAUDE.md`** (코드 폴더, git tracked) = Dify 원본 컨트리뷰터 가이드 (CC BY 4.0, 수정 금지)
- **`.claude/CLAUDE.md`** (옵시디언 junction, gitignored) = spx fork overrides — 원본의 일부 규칙을 덮어씀

## 진입점

| 문서 | 역할 | 언제 보나 |
|------|------|----------|
| [[docs/scope-mapping]] | Dify 챕터 × spx-agent 유무 × 액션 3-way 매핑 | 챕터 작업 결정 시 |
| [[docs/progress]] | Phase 진척 + 챕터별 상태 | 매 작업 전후 |
| [[docs/conventions]] | 번역 톤 + 용어집 + MDX 작성 규칙 | 챕터 작성 시 필독 |
| [[docs/decisions]] | 주요 결정 박제 (호스팅·범위·톤 등) | 충돌·재논의 시 |
| [[docs/references/dify-docs-structure]] | 원본 docs.json + en/ 트리 분석 | Phase 1 분석 시 |
| [[docs/references/ibm-research]] | 5/28 회의 "IBM 문서 참조" 후속 조사 | 필요 시 |

## skills/ (Dify 원본 보존)

Dify가 박아둔 Claude Code 작성 보조 스킬 9종 (dify-docs-api-reference, dify-docs-format-check-cjk, dify-docs-guides 등). 우리 번역·작성 작업에 활용 가능. 폴더 위치(`.claude/skills/`)가 Claude Code 자동 인식 경로라 별도 등록 불필요.

## 배경 (5/28 + 5/29 회의)

### 5/28 이사님 발언 — 매뉴얼 컨셉
- HTML 매뉴얼 (화면 + 화면 설명)
- 로그 항목 설명 / 대시보드 의미 설명
- 화려한 그림보다 **의미·업무 활용** 초점
- 지식/에이전트 생성 절차 시각화
- Claude Code로 PPT 화면 만들어 설계안 확정
- dify 기본 설명법, **교육용**
- IBM 문서 참조 후보

### 5/29 이사님 발언 — 포팅 지시 (핵심)
> dify docs 포팅해서 spx-agent에 없는 내용 빼고 추가한 항목들 추가하기, **use dify 만**, 영어로 포팅해서 한국어 자연스럽게 번역하기

## 작업 흐름 (7 Phase)

```
Phase 0: 환경 셋업 (mintlify dev 동작 확인)
Phase 1: 소스 분석 + 범위 결정 (scope-mapping 작성)
Phase 2: 파일럿 1챕터 (형식 확정)
Phase 3: 이사님 컨펌 (형식·범위·호스팅) — 호스팅: Docusaurus 확정 (2026-06-02)
Phase 3.5: Mintlify → Docusaurus 전환 실행 (~4~5.5일)
Phase 4: 본격 포팅 (반복 작업)
Phase 5: 화면/스크린샷 교체
Phase 6: 빌드·호스팅·배포 (사내 nginx)
```

## 기술 스택

- 빌드: **Docusaurus** (Phase 3.5에서 Mintlify에서 전환 — [[docs/decisions]] 2026-06-02)
  - Phase 0~3 산출물은 Mintlify 기반(`docs.json` + MDX), Phase 3.5에서 `sidebars.js` + Docusaurus MDX v3로 변환
  - MDXComponents 글로벌 래퍼로 Mintlify 컴포넌트(`<Info>`, `<Frame>` 등 14종) 호환 처리
- 검색: `@easyops-cn/docusaurus-search-local` (한국어, 오프라인)
- 로컬: `npm run start` (Docusaurus dev server)
- Node 22 (회사 표준)
- 호스팅: 사내 nginx (Phase 6) — Mintlify Cloud/Enterprise 옵션 폐기

## 응답 규칙

- 한국어 자연스럽게 (직역 X, 5/29 이사님 지시)
- 전문 용어는 [[conventions]] 용어집 참조 (일관성 유지)
