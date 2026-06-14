# モックパターン

外部依存のモックに関する言語非依存の原則と、フレームワーク別の具体例。

---

## 言語非依存の原則

### 何をモックするか

テストで制御できない・遅い・外部への副作用を持つものをモックする:

| 依存の種類 | モックする理由 |
|-----------|--------------|
| データベース（DB） | 状態の保持・遅延・副作用を排除する |
| キャッシュ（Redis 等） | テスト環境での可用性を保証できない |
| 外部 HTTP API | レート制限・ネットワーク遅延・コストを排除する |
| LLM クライアント（OpenAI 等） | コスト・非決定性・遅延を排除する |
| 時刻 / 現在時刻 | テストが時刻依存にならないようにする |
| 乱数生成 | テストの再現性を保証する |
| ファイルシステム | 副作用を排除しポータブルにする |

### 何をモックしないか

- テスト対象のユニットそのもの
- 純粋な関数や計算ロジック
- 同パッケージ内の小さなヘルパー（テスト対象から近い）
- テスト用 DB（インメモリ SQLite 等）を使う統合テスト

### テストダブルの種類

| 種類 | 説明 | 使い所 |
|------|------|--------|
| **Stub** | 固定値を返す単純な代替品 | 戻り値だけが重要なとき |
| **Mock** | 呼び出しを記録・検証できる | 呼び出し回数・引数の検証が必要なとき |
| **Spy** | 実装をラップして呼び出しを追跡する | 既存実装を保ちながら呼び出しを確認したいとき |
| **Fake** | シンプルな実装を持つ代替品（例: インメモリ DB） | 統合テストで依存を差し替えるとき |

### 決定論性の確保

テストは同じ入力に対して常に同じ結果を返さなければならない:
- 現在時刻は固定値に差し替える
- 乱数シードを固定する、またはモックする
- 外部 API レスポンスは固定のフィクスチャで代替する

---

## JS/TS: Jest / Vitest のモックパターン

### Supabase のモック

```typescript
// Supabase クライアントを完全にモック
jest.mock('@/lib/supabase', () => ({
  supabase: {
    from: jest.fn(() => ({
      select: jest.fn(() => ({
        eq: jest.fn(() => Promise.resolve({
          data: [{ id: 1, name: 'テストマーケット' }],
          error: null
        }))
      }))
    }))
  }
}))
```

### Redis のモック

```typescript
// Redis クライアントのメソッドをモック
jest.mock('@/lib/redis', () => ({
  searchMarketsByVector: jest.fn(() => Promise.resolve([
    { slug: 'test-market', similarity_score: 0.95 }
  ])),
  checkRedisHealth: jest.fn(() => Promise.resolve({ connected: true }))
}))
```

### OpenAI（LLM）のモック

```typescript
// LLM クライアントをモックして埋め込みを固定する
jest.mock('@/lib/openai', () => ({
  generateEmbedding: jest.fn(() => Promise.resolve(
    new Array(1536).fill(0.1) // 1536次元の固定埋め込みベクトル
  ))
}))
```

### 時刻のモック（Jest）

```typescript
// 現在時刻を固定する
jest.useFakeTimers()
jest.setSystemTime(new Date('2025-01-01T00:00:00Z'))

// テスト後に元に戻す
afterAll(() => {
  jest.useRealTimers()
})
```

### HTTP API のモック（MSW）

```typescript
// msw を使ったサービスワーカーベースのモック
import { rest } from 'msw'
import { setupServer } from 'msw/node'

const server = setupServer(
  rest.get('https://api.example.com/data', (req, res, ctx) => {
    return res(ctx.json({ items: ['a', 'b', 'c'] }))
  })
)

beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

---

## Python: pytest のモックパターン

### `unittest.mock` を使った基本モック

```python
from unittest.mock import patch, MagicMock

def test_get_market_calls_db(monkeypatch):
    """DB を呼ぶ関数をモックする"""
    mock_db = MagicMock()
    mock_db.query.return_value = [{"id": 1, "name": "テストマーケット"}]

    monkeypatch.setattr("myapp.services.market_service.db", mock_db)

    from myapp.services.market_service import get_market
    result = get_market(1)

    mock_db.query.assert_called_once()
    assert result["name"] == "テストマーケット"
```

### `monkeypatch` による HTTP リクエストのモック

```python
import pytest
import requests

def test_external_api_call(monkeypatch):
    """外部 API への HTTP リクエストをモックする"""
    class MockResponse:
        status_code = 200
        def json(self):
            return {"items": ["a", "b"]}

    monkeypatch.setattr(requests, "get", lambda url, **kwargs: MockResponse())

    from myapp.services.external_service import fetch_items
    result = fetch_items()
    assert result == ["a", "b"]
```

### pytest fixtures を使ったモック

```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.fixture
def mock_llm_client():
    """LLM クライアントのフィクスチャ"""
    with patch("myapp.services.llm_service.client") as mock_client:
        mock_client.embeddings.create = AsyncMock(
            return_value=type("obj", (object,), {
                "data": [type("e", (object,), {"embedding": [0.1] * 1536})()]
            })()
        )
        yield mock_client

@pytest.mark.asyncio
async def test_generate_embedding(mock_llm_client):
    """LLM クライアントがモックされた状態で埋め込みを生成する"""
    from myapp.services.llm_service import generate_embedding
    result = await generate_embedding("テスト")
    assert len(result) == 1536
    mock_llm_client.embeddings.create.assert_called_once()
```

### 時刻のモック

```python
from unittest.mock import patch
from datetime import datetime

def test_time_dependent_logic():
    """現在時刻を固定してテストする"""
    fixed_time = datetime(2025, 1, 1, 0, 0, 0)
    with patch("myapp.services.event_service.datetime") as mock_dt:
        mock_dt.now.return_value = fixed_time
        from myapp.services.event_service import is_event_active
        result = is_event_active()
        # 固定時刻での結果を検証する
```

---

## Go: インターフェース + testify/mock

### インターフェース定義（モックの前提）

```go
// repository.go — インターフェースを定義してモック可能にする
package repository

type MarketRepository interface {
    FindByID(id int) (*Market, error)
    List() ([]Market, error)
    Create(m *Market) error
}
```

### testify/mock を使ったモック実装

```go
// mock_market_repository.go
package mocks

import (
    "myapp/repository"
    "github.com/stretchr/testify/mock"
)

type MockMarketRepository struct {
    mock.Mock
}

func (m *MockMarketRepository) FindByID(id int) (*repository.Market, error) {
    args := m.Called(id)
    return args.Get(0).(*repository.Market), args.Error(1)
}

func (m *MockMarketRepository) List() ([]repository.Market, error) {
    args := m.Called()
    return args.Get(0).([]repository.Market), args.Error(1)
}

func (m *MockMarketRepository) Create(market *repository.Market) error {
    args := m.Called(market)
    return args.Error(0)
}
```

### モックを使ったテスト

```go
// market_service_test.go
package services_test

import (
    "testing"
    "myapp/mocks"
    "myapp/repository"
    "myapp/services"
    "github.com/stretchr/testify/assert"
)

func TestGetMarket_Success(t *testing.T) {
    // モックをセットアップ
    mockRepo := new(mocks.MockMarketRepository)
    expected := &repository.Market{ID: 1, Name: "テストマーケット"}
    mockRepo.On("FindByID", 1).Return(expected, nil)

    // テスト対象にモックを注入
    svc := services.NewMarketService(mockRepo)
    result, err := svc.GetMarket(1)

    // 検証
    assert.NoError(t, err)
    assert.Equal(t, "テストマーケット", result.Name)
    mockRepo.AssertExpectations(t) // モックが期待通り呼ばれたか確認
}

func TestGetMarket_DBError(t *testing.T) {
    mockRepo := new(mocks.MockMarketRepository)
    mockRepo.On("FindByID", 99).Return((*repository.Market)(nil), fmt.Errorf("DB接続エラー"))

    svc := services.NewMarketService(mockRepo)
    _, err := svc.GetMarket(99)

    assert.Error(t, err)
    assert.Contains(t, err.Error(), "DB接続エラー")
}
```

### HTTP クライアントのモック（インターフェース版）

```go
// http_client.go — インターフェース定義
type HTTPClient interface {
    Get(url string) (*http.Response, error)
}

// テスト用スタブ
type MockHTTPClient struct {
    response *http.Response
    err      error
}

func (m *MockHTTPClient) Get(url string) (*http.Response, error) {
    return m.response, m.err
}
```
