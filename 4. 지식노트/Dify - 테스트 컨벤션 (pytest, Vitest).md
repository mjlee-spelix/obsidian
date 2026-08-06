---
tags: [dify, 개발, 테스트]
date: 2026-04-29
---
# Dify - 테스트 컨벤션 (pytest, Vitest)

## 백엔드 (pytest)

### 폴더 구조

```
api/tests/
├── conftest.py                          ← workflow 런타임 바인딩
├── unit_tests/
│   ├── conftest.py                      ← Flask 앱, Redis 모킹, SQLite in-memory
│   ├── configs/
│   ├── core/
│   └── ...                              ← 소스 구조 반영
└── integration_tests/
    ├── conftest.py                      ← 실제 Flask 앱, 실제 DB, 테스트 계정
    └── controllers/console/app/         ← 컨트롤러 경로 반영
```

### 네이밍 패턴

- 파일: `test_*.py`
- 클래스 기반: `class TestFeedbackApiBasic:` + `def test_*(self, ...)`
- 함수 기반: `def test_dify_config(monkeypatch):`

### conftest / fixture 패턴

**단위 테스트 conftest** (SQLite in-memory):
```python
# autouse=True로 테스트마다 자동 리셋
@pytest.fixture(autouse=True)
def _reset_redis(redis_mock):
    redis_mock.reset_mock()

# 헬퍼 함수
setup_mock_tenant_account_query()
setup_mock_dataset_tenant_query()
```

**통합 테스트 conftest** (실제 DB):
```python
@pytest.fixture(scope="session")
def dify_config():
    ...

@pytest.fixture
def auth_header():
    # JWT 토큰 포함된 인증 헤더 반환

@pytest.fixture
def test_client():
    # FlaskClient 반환
```

### mock 패턴

```python
from unittest.mock import MagicMock, patch

# Redis 모킹
redis_mock = MagicMock()
patch.object(ext_redis, "redis_client", redis_mock)

# 환경변수 모킹
monkeypatch.setenv("DB_TYPE", "postgresql")
```

### 실행 방법

```bash
pytest api/tests/unit_tests/
pytest api/tests/integration_tests/
```

### pytest 설정 (pytest.ini)

- `pythonpath = .`
- 커버리지: JSON/XML 리포트
- importlib 모드
- 테스트용 환경변수 (API 키, 엔드포인트) 정의

---

## 프론트엔드 (Vitest)

### 프레임워크

- **Vitest** (Vite 기반) + **React Testing Library** + **Happy-DOM**

### 파일 위치/네이밍

```
web/app/components/기능명/__tests__/index.spec.tsx
```

- `__tests__/` 폴더 안에 `*.spec.tsx`

### 테스트 패턴

```typescript
import type { Mock } from 'vitest'

vi.mock('@/context/provider-context', () => ({
  useProviderContext: vi.fn(),
}))

describe('AddAnnotationModal', () => {
  const baseProps = { isShow: true, onHide: vi.fn(), onAdd: vi.fn() }

  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('should render', () => { ... })
})
```

### 설정 (vite.config.ts)

```javascript
test: {
  pool: 'threads',
  environment: 'happy-dom',
  globals: true,
  setupFiles: ['./vitest.setup.ts'],
  coverage: { provider: 'v8' }
}
```

### 실행 방법

```bash
cd web
pnpm test              # 테스트 실행
pnpm test:coverage     # 커버리지
pnpm test:watch        # 감시 모드
```

## 관련 노트

- [[4. 지식노트/Dify - 새 API 엔드포인트 등록 방법.md]]
- [[4. 지식노트/Dify - 앱 통계 UI 구조.md]]
