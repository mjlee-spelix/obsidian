```
[public 스키마] (총 ~90개 테이블)
│
├── tenants                                   ← 워크스페이스 루트
│   │
│   ├── ─── [계정·권한] ───
│   │   ├── tenant_account_joins              ← 사용자-워크스페이스 매핑
│   │   ├── account_plugin_permissions        ← 플러그인 설치 권한
│   │   └── tenant_plugin_auto_upgrade_strategies
│   │
│   ├── ─── [모델 프로바이더] ───
│   │   ├── providers                         ← 프로바이더 자격증명
│   │   ├── provider_models                   ← 모델 목록
│   │   ├── provider_credentials              ← 커스텀 프로바이더 인증
│   │   ├── provider_model_credentials        ← 모델별 인증
│   │   ├── provider_model_settings           ← 모델별 설정
│   │   ├── load_balancing_model_configs      ← 로드밸런싱 설정
│   │   ├── tenant_default_models             ← 기본 모델 설정
│   │   └── tenant_preferred_model_providers  ← 선호 프로바이더
│   │
│   ├── ─── [앱] apps ★ ───
│   │   ├── app_model_configs                 ← Chat/Completion 앱 프롬프트 설정
│   │   ├── sites                             ← WebApp 공개 설정
│   │   ├── api_tokens                        ← API 키
│   │   ├── app_mcp_servers                   ← MCP 서버 연결
│   │   ├── app_annotation_settings           ← 어노테이션 설정
│   │   ├── trace_app_config                  ← 트레이싱 설정
│   │   │
│   │   ├── ── [채팅] conversations ★ ──
│   │   │   └── messages ★                   ← (유일한 DB-level FK)
│   │   │       ├── message_feedbacks         ← 좋아요/싫어요
│   │   │       ├── message_files             ← 첨부파일
│   │   │       ├── message_annotations       ← 어노테이션
│   │   │       ├── message_chains            ← Agent 실행 체인
│   │   │       └── message_agent_thoughts ★  ← Agent LLM 호출 상세·토큰
│   │   │
│   │   ├── ── [워크플로우 정의] workflows ★ ──
│   │   │   ├── workflow_draft_variables      ← 디버깅 중 변수 상태
│   │   │   │   └── workflow_draft_variable_files
│   │   │   │
│   │   │   └── ── [실행] workflow_runs ★ ──
│   │   │       ├── workflow_node_executions ★ ← 노드별 토큰·메타데이터
│   │   │       │   └── workflow_node_execution_offload ← 대용량 I/O 외부저장
│   │   │       ├── workflow_app_logs          ← 실행 로그 (보존기간 내)
│   │   │       ├── workflow_archive_logs ★    ← 실행 로그 (아카이브, 통계용)
│   │   │       ├── workflow_trigger_logs      ← 트리거 실행 로그
│   │   │       ├── workflow_pauses            ← HITL 일시정지 상태
│   │   │       │   └── workflow_pause_reasons
│   │   │       └── workflow_conversation_variables ← 대화 변수 현재값
│   │   │
│   │   ├── ── [트리거] ──
│   │   │   ├── app_triggers
│   │   │   ├── trigger_subscriptions
│   │   │   ├── workflow_webhook_triggers
│   │   │   ├── workflow_plugin_triggers
│   │   │   └── workflow_schedule_plans
│   │   │
│   │   └── tool_workflow_providers            ← 앱을 도구로 게시
│   │
│   ├── ─── [데이터셋] datasets ★ ───
│   │   ├── dataset_process_rules             ← 청킹 규칙
│   │   ├── dataset_keyword_tables            ← 키워드 인덱스
│   │   ├── dataset_queries                   ← 검색 쿼리 로그
│   │   ├── dataset_permissions               ← 부분공개 권한 (account별)
│   │   ├── dataset_metadatas                 ← 메타데이터 스키마 정의
│   │   │   └── dataset_metadata_bindings     ← 문서별 메타데이터 값
│   │   ├── external_knowledge_bindings  ──→  external_knowledge_apis
│   │   ├── pipelines                         ← RAG 파이프라인
│   │   │   └── document_pipeline_execution_logs
│   │   │
│   │   └── documents
│   │       └── document_segments
│   │           ├── child_chunks              ← 계층형 청킹 하위 청크
│   │           ├── document_segment_summaries
│   │           └── segment_attachment_bindings  ──→  upload_files
│   │
│   ├── ─── [도구] ───
│   │   ├── tool_builtin_providers            ← 빌트인 도구 자격증명
│   │   ├── tool_api_providers                ← OpenAPI 도구
│   │   │   └── tool_label_bindings
│   │   ├── tool_mcp_providers                ← MCP 도구
│   │   ├── tool_oauth_tenant_clients         ← OAuth 클라이언트 (테넌트)
│   │   └── tool_published_apps               ← 앱을 도구로 게시
│   │
│   ├── ─── [태그] tags ★ ───
│   │   └── tag_bindings  ──→  apps / datasets  (target_id로 구분)
│   │
│   ├── tenant_credit_pools                   ← 크레딧 잔액
│   └── tidb_auth_bindings                    ← TiDB Serverless 연동
│
├── accounts                                  ← 사용자 계정 (tenants와 독립)
│   ├── account_integrates                    ← Google 등 OAuth 연동
│   └── tenant_account_joins                  ← ↑ tenants와 교차
│
├── ─── [N:M 교차 테이블] ───
│   └── app_dataset_joins                     ← apps ↔ datasets
│
├── ─── [공유 리소스] ───
│   ├── upload_files                          ← 업로드 파일 (전역)
│   ├── embeddings                            ← 임베딩 캐시 (모델+해시 키)
│   ├── dataset_collection_bindings           ← 벡터DB 컬렉션 매핑
│   ├── external_knowledge_apis               ← 외부 지식베이스 API
│   ├── tool_oauth_system_clients             ← OAuth 클라이언트 (시스템)
│   ├── tool_files                            ← 도구 출력 파일
│   ├── tool_model_invokes                    ← 도구 모델 호출 로그
│   ├── tool_conversation_variables           ← 도구 대화 변수
│   ├── execution_extra_contents              ← 실행 추가 콘텐츠
│   ├── human_input_forms                     ← HITL 입력 폼 정의
│   │   ├── human_input_form_deliveries
│   │   └── human_input_form_recipients
│   └── dataset_retriever_resources           ← RAG 검색 결과 기록
│
├── ─── [인프라·시스템] ───
│   ├── dify_setups                           ← 초기 설정 완료 플래그
│   ├── invitation_codes                      ← 초대 코드
│   ├── operation_logs                        ← 관리자 작업 로그
│   ├── api_requests                          ← API 요청 이력
│   ├── rate_limit_logs                       ← Rate limit 이벤트
│   ├── dataset_auto_disable_logs             ← 자동 비활성화 이벤트
│   ├── whitelists                            ← IP/도메인 화이트리스트
│   ├── celery_taskmeta                       ← Celery 태스크 결과
│   ├── celery_tasksetmeta                    ← Celery 그룹 결과
│   ├── datasource_oauth_params               ← 데이터소스 OAuth 파라미터
│   ├── datasource_providers                  ← 데이터소스 프로바이더
│   ├── datasource_oauth_tenant_params
│   ├── trigger_oauth_system_clients
│   └── trigger_oauth_tenant_clients
│
└── ─── [SaaS 전용·기타] ───
    ├── end_users                             ← WebApp 익명 사용자
    ├── saved_messages                        ← WebApp 저장 메시지
    ├── pinned_conversations                  ← WebApp 고정 대화
    ├── app_annotation_hit_histories          ← 어노테이션 히트 이력
    ├── recommended_apps                      ← 탐색 페이지 추천 앱
    ├── installed_apps                        ← 설치된 앱
    ├── trial_apps / account_trial_app_records
    ├── exporle_banners                       ← 탐색 페이지 배너 (오타 그대로)
    ├── oauth_provider_apps
    ├── pipeline_built_in_templates           ← 파이프라인 기본 템플릿
    ├── pipeline_customized_templates         ← 사용자 정의 템플릿
    ├── pipeline_recommended_plugins
    ├── provider_orders                       ← 프로바이더 우선순위
    └── api_based_extensions                  ← API 기반 확장

```