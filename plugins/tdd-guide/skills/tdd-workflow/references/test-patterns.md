# テストパターン具体例

ユニット・統合・E2E のテストパターンを JS/TS・Python・Go で示す。

---

## ユニットテストパターン

### JS/TS（Jest / Vitest + Testing Library）

```typescript
// Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

describe('Button コンポーネント', () => {
  it('正しいテキストでレンダリングされる', () => {
    render(<Button>クリック</Button>)
    expect(screen.getByText('クリック')).toBeInTheDocument()
  })

  it('クリック時に onClick が呼ばれる', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>クリック</Button>)
    fireEvent.click(screen.getByRole('button'))
    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('disabled prop が true のとき無効化される', () => {
    render(<Button disabled>クリック</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })
})
```

### Python（pytest）

```python
# test_button.py
import pytest
from myapp.components import Button

class TestButton:
    def test_renders_with_correct_text(self):
        """正しいテキストプロパティを持つ"""
        btn = Button(text="クリック")
        assert btn.text == "クリック"

    def test_calls_on_click_when_clicked(self):
        """クリック時にコールバックが呼ばれる"""
        clicked = []
        btn = Button(text="クリック", on_click=lambda: clicked.append(True))
        btn.click()
        assert len(clicked) == 1

    def test_is_disabled_when_disabled_prop_is_true(self):
        """disabled=True のとき無効化される"""
        btn = Button(text="クリック", disabled=True)
        assert btn.is_disabled is True
```

### Go（testing + table-driven）

```go
// button_test.go
package ui_test

import (
    "testing"
    "myapp/ui"
)

func TestButton(t *testing.T) {
    tests := []struct {
        name     string
        text     string
        disabled bool
        wantText string
    }{
        {name: "通常ボタン", text: "クリック", disabled: false, wantText: "クリック"},
        {name: "無効ボタン", text: "送信", disabled: true, wantText: "送信"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            btn := ui.NewButton(tt.text, tt.disabled)
            if btn.Text != tt.wantText {
                t.Errorf("Text = %q, want %q", btn.Text, tt.wantText)
            }
            if btn.Disabled != tt.disabled {
                t.Errorf("Disabled = %v, want %v", btn.Disabled, tt.disabled)
            }
        })
    }
}
```

---

## 統合テストパターン

### JS/TS（Next.js route / API ルート）

```typescript
// markets/route.test.ts
import { NextRequest } from 'next/server'
import { GET } from './route'

describe('GET /api/markets', () => {
  it('正常時にマーケット一覧を返す', async () => {
    const request = new NextRequest('http://localhost/api/markets')
    const response = await GET(request)
    const data = await response.json()

    expect(response.status).toBe(200)
    expect(data.success).toBe(true)
    expect(Array.isArray(data.data)).toBe(true)
  })

  it('不正なクエリパラメータを検証する', async () => {
    const request = new NextRequest('http://localhost/api/markets?limit=invalid')
    const response = await GET(request)
    expect(response.status).toBe(400)
  })

  it('データベースエラーを適切にハンドリングする', async () => {
    // DB 障害をモックする（mocking-patterns.md 参照）
    const request = new NextRequest('http://localhost/api/markets')
    const response = await GET(request)
    expect(response.status).toBe(500)
  })
})
```

### Python（FastAPI + TestClient）

```python
# test_markets_api.py
import pytest
from fastapi.testclient import TestClient
from myapp.main import app

client = TestClient(app)

class TestMarketsAPI:
    def test_get_markets_returns_list(self):
        """正常時にマーケット一覧を返す"""
        response = client.get("/api/markets")
        assert response.status_code == 200
        data = response.json()
        assert data["success"] is True
        assert isinstance(data["data"], list)

    def test_invalid_query_param_returns_400(self):
        """不正なクエリパラメータで 400 を返す"""
        response = client.get("/api/markets?limit=invalid")
        assert response.status_code == 422  # FastAPI のバリデーションエラー

    def test_db_error_returns_500(self, monkeypatch):
        """DB エラー時に 500 を返す（mocking-patterns.md 参照）"""
        def mock_get_markets():
            raise Exception("DB connection failed")
        monkeypatch.setattr("myapp.services.market_service.get_markets", mock_get_markets)
        response = client.get("/api/markets")
        assert response.status_code == 500
```

### Go（net/http + httptest）

```go
// markets_handler_test.go
package handlers_test

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "myapp/handlers"
)

func TestGetMarkets(t *testing.T) {
    t.Run("正常時にマーケット一覧を返す", func(t *testing.T) {
        req := httptest.NewRequest(http.MethodGet, "/api/markets", nil)
        w := httptest.NewRecorder()

        handlers.GetMarkets(w, req)

        if w.Code != http.StatusOK {
            t.Errorf("status = %d, want %d", w.Code, http.StatusOK)
        }
        var resp map[string]interface{}
        json.NewDecoder(w.Body).Decode(&resp)
        if _, ok := resp["data"]; !ok {
            t.Error("response should contain 'data' field")
        }
    })

    t.Run("不正なクエリパラメータで 400 を返す", func(t *testing.T) {
        req := httptest.NewRequest(http.MethodGet, "/api/markets?limit=invalid", nil)
        w := httptest.NewRecorder()
        handlers.GetMarkets(w, req)
        if w.Code != http.StatusBadRequest {
            t.Errorf("status = %d, want %d", w.Code, http.StatusBadRequest)
        }
    })
}
```

---

## E2E テストパターン（Playwright）

### JS/TS（@playwright/test）

```typescript
// e2e/markets.spec.ts
import { test, expect } from '@playwright/test'

test('ユーザーがマーケットを検索・フィルタリングできる', async ({ page }) => {
  await page.goto('/')
  await page.click('a[href="/markets"]')
  await expect(page.locator('h1')).toContainText('マーケット')

  // 検索
  await page.fill('input[placeholder="マーケットを検索"]', 'テスト')
  await page.waitForTimeout(600) // デバウンス待機

  // 結果を確認
  const results = page.locator('[data-testid="market-card"]')
  await expect(results).toHaveCount(5, { timeout: 5000 })

  // フィルタリング
  await page.click('button:has-text("アクティブ")')
  await expect(results).toHaveCount(3)
})

test('ユーザーが新しいマーケットを作成できる', async ({ page }) => {
  await page.goto('/creator-dashboard')
  await page.fill('input[name="name"]', 'テストマーケット')
  await page.fill('textarea[name="description"]', 'テスト説明文')
  await page.fill('input[name="endDate"]', '2025-12-31')
  await page.click('button[type="submit"]')
  await expect(page.locator('text=マーケットが作成されました')).toBeVisible()
  await expect(page).toHaveURL(/\/markets\/test-market/)
})
```

### Python（playwright-python）

```python
# test_e2e_markets.py
import pytest
from playwright.sync_api import Page, expect

def test_user_can_search_markets(page: Page):
    """ユーザーがマーケットを検索できる"""
    page.goto("/")
    page.click("a[href='/markets']")
    expect(page.locator("h1")).to_contain_text("マーケット")

    page.fill("input[placeholder='マーケットを検索']", "テスト")
    page.wait_for_timeout(600)

    results = page.locator("[data-testid='market-card']")
    expect(results).to_have_count(5, timeout=5000)

def test_user_can_create_market(page: Page):
    """ユーザーが新しいマーケットを作成できる"""
    page.goto("/creator-dashboard")
    page.fill("input[name='name']", "テストマーケット")
    page.fill("textarea[name='description']", "テスト説明文")
    page.click("button[type='submit']")
    expect(page.locator("text=マーケットが作成されました")).to_be_visible()
```

---

## テストファイル構成例

### JS/TS（Next.js）

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx          # ユニットテスト
│   │   └── Button.stories.tsx
│   └── MarketCard/
│       ├── MarketCard.tsx
│       └── MarketCard.test.tsx
├── app/
│   └── api/
│       └── markets/
│           ├── route.ts
│           └── route.test.ts        # 統合テスト
└── e2e/
    ├── markets.spec.ts              # E2E テスト
    ├── trading.spec.ts
    └── auth.spec.ts
```

### Python（FastAPI）

```
myapp/
├── api/
│   └── markets.py
├── services/
│   └── market_service.py
tests/
├── unit/
│   ├── test_market_service.py
│   └── test_utils.py
├── integration/
│   └── test_markets_api.py
└── e2e/
    └── test_e2e_markets.py
```

### Go

```
myapp/
├── handlers/
│   ├── markets.go
│   └── markets_test.go              # 統合テスト（同パッケージ or _test）
├── services/
│   ├── market.go
│   └── market_test.go               # ユニットテスト
└── e2e/
    └── markets_e2e_test.go
```

---

## よくある誤り（アンチパターン）と正しい書き方

### 実装詳細をテストする（NG）→ 振る舞いをテストする（OK）

```typescript
// NG: 内部状態をテストする
expect(component.state.count).toBe(5)

// OK: ユーザーが見る内容をテストする
expect(screen.getByText('カウント: 5')).toBeInTheDocument()
```

### 脆弱なセレクタ（NG）→ セマンティックセレクタ（OK）

```typescript
// NG: CSSクラスは変わりやすい
await page.click('.css-class-xyz')

// OK: 意味的に安定したセレクタ
await page.click('button:has-text("送信")')
await page.click('[data-testid="submit-button"]')
```

### テスト間の依存（NG）→ 独立したテスト（OK）

```typescript
// NG: テストが前のテストに依存する
test('ユーザーを作成する', () => { /* ... */ })
test('同じユーザーを更新する', () => { /* 前のテストに依存 */ })

// OK: 各テストが自分でデータをセットアップする
test('ユーザーを作成する', () => {
  const user = createTestUser()
  // テストロジック
})
test('ユーザーを更新する', () => {
  const user = createTestUser() // 独立してセットアップ
  // 更新ロジック
})
```
