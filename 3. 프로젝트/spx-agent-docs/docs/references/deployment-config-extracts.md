# 배포 설정 추출 (deployment-config-extracts)

> 전역 규칙 #4에 따라 본문에서 빼고 별도 수집한 환경 변수·배포 설정.
> 포팅 중 발견 시 즉시 추가. 추후 일괄 검토 후 노출 여부 결정.
> 작성: 2026-06-09

## 수집된 항목

### Logs — 워크플로우 로그 보관 정책 (Monitor/Logs, 2026-06-09)

원본: `en/use-dify/monitor/logs.mdx` "Log Retention" 섹션

| 환경 변수 | 역할 | 원본 설명 |
|----------|------|----------|
| `WORKFLOW_LOG_CLEANUP_ENABLED` | 워크플로우 로그 자동 정리 활성화 | Self-hosted: Unlimited by default; configurable via this env var |
| `WORKFLOW_LOG_RETENTION_DAYS` | 로그 보관 일수 | 보관 기간 설정 |
| `WORKFLOW_LOG_CLEANUP_BATCH_SIZE` | 정리 배치 크기 | 한 번에 삭제하는 로그 수 |

**추출 사유**: SaaS 플랜별 보관 정책(Sandbox 30일, Pro/Team 무제한) 삭제 시 Self-hosted env var도 함께 제거. spx-agent는 CE 기반이라 이 env var가 유효할 수 있으나, 사용자 문서 본문에 넣기엔 시스템 관리 영역. 추후 관리자 가이드 챕터로 묶을지 검토.

### Knowledge — 파일 업로드 한도 (create-knowledge/import-text-data, 2026-06-10)

원본: `en/use-dify/knowledge/create-knowledge/import-text-data/readme.mdx`

| 환경 변수 | 역할 | 원본 설명 |
|----------|------|----------|
| `UPLOAD_FILE_SIZE_LIMIT` | 업로드 파일 최대 크기 | 기본 15 MB. self-hosted에서 조정 가능 |
| `UPLOAD_FILE_BATCH_LIMIT` | 1회 업로드 최대 파일 수 | 기본 5. self-hosted에서 조정 가능 |
| `ATTACHMENT_IMAGE_FILE_SIZE_LIMIT` | 청크 첨부 이미지 최대 크기 | 기본 2 MB |
| `SINGLE_CHUNK_ATTACHMENT_LIMIT` | 청크당 최대 첨부파일 수 | 기본 10 |

**추출 사유**: 전역 규칙 #4 — self-hosted 분기 `<Tip>` 2개에서 env var 명칭 추출. 사용자 본문에서는 한도 수치만 안내(5개, 15 MB, 2 MB, 10개), env var 조정 방법은 시스템 관리 영역.

### Code — 셀프 호스팅 샌드박스 기동 (nodes/code, 2026-06-15)

원본: `en/use-dify/nodes/code.mdx` "Self-Hosted Setup" 섹션 (en L118-126)

| 항목 | 내용 |
|------|------|
| 명령 | `docker-compose -f docker-compose.middleware.yaml up -d` |
| 요구 | Docker 필요. 안전한 코드 실행을 위해 메인 시스템과 격리된 샌드박스 서비스 기동 |

원문 요지: self-hosted Dify에서는 코드 노드의 안전한 실행을 위해 샌드박스 서비스를 기동해야 하며, 이 서비스는 Docker를 요구하고 코드 실행을 메인 시스템에서 격리한다.

**추출 사유**: 전역 규칙 #4(self-hosted 배포 설정) — docker-compose 기동 명령은 시스템 관리 영역. 사용자 본문에서는 샌드박스에 의존성이 사전 설치돼 있다는 안내만 유지하고, 기동 절차는 관리자 가이드로 분리 검토.

### Doc 추출기 — Unstructured API 외부 의존성 (nodes/doc-extractor, 2026-06-15)

원본: `en/use-dify/nodes/doc-extractor.mdx` "External Dependencies" 섹션 (en L92-97)

| 환경 변수 | 역할 |
|----------|------|
| `UNSTRUCTURED_API_URL` | Unstructured API 서비스 엔드포인트 |
| `UNSTRUCTURED_API_KEY` | Unstructured API 인증 키 |

원문 요지: 일부 형식(DOC 레거시 Word, PowerPoint, EPUB 등 API 처리 사용 시)은 `UNSTRUCTURED_API_URL`과 `UNSTRUCTURED_API_KEY`로 구성된 **Unstructured API** 서비스가 필요하다.

**추출 사유**: 전역 규칙 #4 — 외부 서비스 연동 env var는 시스템 관리 영역. 사용자 본문에는 지원 형식만 안내하고, 외부 의존성 구성은 관리자 가이드로 분리 검토.

### 템플릿 — 출력 길이 한도 (nodes/template, 2026-06-15)

원본: `en/use-dify/nodes/template.mdx` "Output Limits" 섹션 (en L105-107)

| 환경 변수 | 역할 | 기본값 |
|----------|------|--------|
| `TEMPLATE_TRANSFORM_MAX_LENGTH` | 템플릿 출력 최대 문자 수 | 80,000자 |

원문 요지: 템플릿 출력은 기본 **80,000자**로 제한되며 `TEMPLATE_TRANSFORM_MAX_LENGTH`로 조정할 수 있다. 대규모 템플릿 출력에서 메모리 문제를 방지하고 합리적인 처리 시간을 보장하기 위함이다.

**추출 사유**: 전역 규칙 #4 — 한도 조정 env var는 시스템 관리 영역. 사용자 본문에는 한도 수치(80,000자)와 그 목적만 안내하고, env var 조정 방법은 관리자 가이드로 분리 검토.
