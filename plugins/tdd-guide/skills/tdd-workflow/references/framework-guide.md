# フレームワーク別ガイド

FW ごとのセットアップ・テスト実行・カバレッジ計測/閾値強制・ウォッチモード・pre-commit・CI/CD の設定。

---

## JS/TS: Jest / Vitest

### セットアップ

```bash
# Jest
npm install --save-dev jest @types/jest ts-jest

# Vitest（Vite プロジェクト向け）
npm install --save-dev vitest
```

### テスト実行

```bash
# 全テストを実行
npm test

# ウォッチモード（開発中はこちら）
npm test -- --watch

# 特定ファイルだけ実行
npm test -- src/components/Button.test.tsx
```

### カバレッジ計測

```bash
npm run test:coverage
```

### カバレッジ閾値の設定（Jest）

`jest.config.js` または `package.json` の `jest` フィールドに追加:

```json
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    },
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.stories.{ts,tsx}",
      "!src/**/*.d.ts"
    ]
  }
}
```

### カバレッジ閾値の設定（Vitest）

`vite.config.ts` に追加:

```typescript
// vite.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      thresholds: {
        branches: 80,
        functions: 80,
        lines: 80,
        statements: 80
      },
      include: ['src/**/*.{ts,tsx}'],
      exclude: ['src/**/*.stories.*', 'src/**/*.d.ts']
    }
  }
})
```

### Pre-commit フック

```bash
# コミット前にテストとリントを実行
# package.json の scripts に追加:
# "precommit": "npm test && npm run lint"
```

または husky を使う:

```bash
npm install --save-dev husky lint-staged
npx husky install
npx husky add .husky/pre-commit "npm test -- --passWithNoTests"
```

### CI/CD（GitHub Actions）

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - name: テストを実行
        run: npm test -- --coverage
      - name: カバレッジをアップロード
        uses: codecov/codecov-action@v3
```

---

## Python: pytest

### セットアップ

```bash
pip install pytest pytest-cov pytest-asyncio

# pyproject.toml または setup.cfg に設定を追加
```

### テスト実行

```bash
# 全テストを実行
pytest

# 詳細出力
pytest -v

# 特定ファイルだけ実行
pytest tests/unit/test_market_service.py

# 特定テストケースだけ実行
pytest tests/unit/test_market_service.py::TestMarketService::test_get_market
```

### ウォッチモード

```bash
pip install pytest-watch

# ファイル変更時に自動再実行
ptw
```

### カバレッジ計測

```bash
# カバレッジを計測しながらテストを実行
pytest --cov=myapp --cov-report=term-missing

# HTML レポートも生成
pytest --cov=myapp --cov-report=html --cov-report=term-missing

# 80% を下回ったら失敗させる
pytest --cov=myapp --cov-fail-under=80
```

### カバレッジ閾値の設定（pyproject.toml）

```toml
[tool.pytest.ini_options]
addopts = "--cov=myapp --cov-report=term-missing --cov-fail-under=80"

[tool.coverage.run]
source = ["myapp"]
omit = ["*/tests/*", "*/migrations/*"]

[tool.coverage.report]
fail_under = 80
show_missing = true
```

### Pre-commit フック

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
```

```bash
pip install pre-commit
pre-commit install
```

### CI/CD（GitHub Actions）

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements-dev.txt
      - name: テストを実行（カバレッジ付き）
        run: pytest --cov=myapp --cov-fail-under=80
      - name: カバレッジをアップロード
        uses: codecov/codecov-action@v3
```

---

## Go: testing / testify

### セットアップ

```bash
# testify をインストール
go get github.com/stretchr/testify

# モックジェネレーターをインストール（任意）
go install github.com/vektra/mockery/v2@latest
```

### テスト実行

```bash
# 全テストを実行
go test ./...

# 詳細出力
go test -v ./...

# 特定パッケージだけ実行
go test ./handlers/...

# 特定テスト関数だけ実行
go test ./services/... -run TestGetMarket

# タイムアウトを設定
go test -timeout 30s ./...
```

### ウォッチモード

```bash
# air などのホットリロードツールを活用するか、entr を使う
find . -name '*.go' | entr go test ./...

# または gotestsum（出力が見やすい）
go install gotest.tools/gotestsum@latest
gotestsum --watch ./...
```

### カバレッジ計測

```bash
# カバレッジ計測（要約）
go test -cover ./...

# カバレッジプロファイルを出力
go test -coverprofile=coverage.out ./...

# HTML レポートで可視化
go tool cover -html=coverage.out -o coverage.html

# カバレッジ率だけ確認
go tool cover -func=coverage.out | tail -1
```

### カバレッジ閾値の強制

Go 標準の `go test` にはビルトインの閾値強制がないため、CI スクリプトで実現する:

```bash
# Makefile の例
.PHONY: test-coverage
test-coverage:
	go test -coverprofile=coverage.out ./...
	@COVERAGE=$$(go tool cover -func=coverage.out | grep total | awk '{print $$3}' | sed 's/%//'); \
	echo "カバレッジ: $${COVERAGE}%"; \
	if [ $$(echo "$${COVERAGE} < 80" | bc) -eq 1 ]; then \
		echo "カバレッジが 80% を下回っています"; \
		exit 1; \
	fi
```

### Pre-commit フック（.githooks/pre-commit）

```bash
#!/bin/bash
set -e

echo "テストを実行中..."
go test ./...
echo "完了"
```

```bash
# フックを登録
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks
```

### CI/CD（GitHub Actions）

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.23'
      - name: 依存関係を確認
        run: go mod tidy
      - name: テストを実行
        run: go test -v -coverprofile=coverage.out ./...
      - name: カバレッジを確認（80%以上を要求）
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
          echo "カバレッジ: ${COVERAGE}%"
          if [ $(echo "${COVERAGE} < 80" | bc) -eq 1 ]; then
            echo "カバレッジが 80% を下回っています"
            exit 1
          fi
      - name: カバレッジをアップロード
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.out
```

---

## フレームワーク比較早見表

| 項目 | JS/TS (Jest/Vitest) | Python (pytest) | Go (testing) |
|------|---------------------|-----------------|--------------|
| **テスト実行** | `npm test` | `pytest` | `go test ./...` |
| **ウォッチモード** | `npm test -- --watch` | `ptw` | `gotestsum --watch` |
| **カバレッジ** | `npm run test:coverage` | `pytest --cov --cov-fail-under=80` | `go test -cover ./...` |
| **閾値設定** | `jest.config.js` / `vite.config.ts` | `pyproject.toml` | CI スクリプト |
| **モックFW** | Jest の `jest.fn()` / Vitest の `vi.fn()` / MSW | `unittest.mock` / `monkeypatch` | インターフェース + `testify/mock` |
| **E2E** | Playwright (`@playwright/test`) | `playwright-python` | `chromedp` 等 |
