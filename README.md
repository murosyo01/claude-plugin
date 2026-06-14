# claude-plugin

claudeのpluginを共通して利用できるようにするためのリポジトリ

個別に導入できます。

## プラグイン一覧

| プラグイン            | 概要                                                                                            | Agent               | Skill                                                                                                                                  | 推奨モデル |
| --------------------- | ----------------------------------------------------------------------------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| **architect**         | システム設計・スケーラビリティ評価・ADR作成                                                     | `architect`         | `architect`                                                                                                                            | Opus       |
| **code-architect**    | 既存コードベース向け実装ブループリント                                                          | `code-architect`    | `code-architect`                                                                                                                       | Sonnet     |
| **code-reviewer**     | セキュリティ・品質・パフォーマンスレビュー                                                      | `code-reviewer`     | `code-review`                                                                                                                          | Sonnet     |
| **code-explorer**     | LSPを活用したコードベース探索・依存関係マッピング                                               | `code-explorer`     | `code-explorer`                                                                                                                        | Opus       |
| **code-simplifier**   | 振る舞いを保ったままコードを簡潔化・可読性向上                                                  | `code-simplifier`   | `code-simplifier`                                                                                                                      | Sonnet     |
| **security-reviewer** | OWASP Top 10・シークレット・インジェクション・SSRFなどセキュリティ脆弱性を専門的に検出          | `security-reviewer` | `security-review`                                                                                                                      | Sonnet     |
| **comment-analyzer**  | コメントの正確性・古さ・過不足・腐敗リスクを分析。ファイルは変更しない                          | `comment-analyzer`  | `comment-analyzer`                                                                                                                     | Sonnet     |
| **network-architect** | エンタープライズ・マルチサイトのネットワーク設計。設定検証・BGP診断・IOS・SSH自動化スキルを同梱 | `network-architect` | `network-config-validation` / `network-bgp-diagnostics` / `network-interface-health` / `cisco-ios-patterns` / `netmiko-ssh-automation` | Sonnet     |

---

## インストール

### マーケットプレイスを登録（初回のみ）

```
/plugin marketplace add murosyo01/murosyo-claude-plugin
```

> ⚠️ このコマンドは GitHub への push 後に利用可能です。ローカル開発中はリポジトリの絶対パスを指定してください：
>
> ```
> /plugin marketplace add /path/to/claude-plugin
> ```

### 個別プラグインのインストール

必要なプラグインだけを選んで導入できます。

```
/plugin install architect@murosyo-claude-plugin
/plugin install code-architect@murosyo-claude-plugin
/plugin install code-reviewer@murosyo-claude-plugin
/plugin install code-explorer@murosyo-claude-plugin
/plugin install code-simplifier@murosyo-claude-plugin
/plugin install security-reviewer@murosyo-claude-plugin
/plugin install comment-analyzer@murosyo-claude-plugin
/plugin install network-architect@murosyo-claude-plugin
```

### 全プラグインを一括インストール

```
/plugin marketplace install murosyo-claude-plugin
```

---

## 各プラグインの使い方

### architect

システム設計の相談・アーキテクチャレビュー・ADR作成を担当します。

**自動起動トリガー例（agent）：**

- 「この決済サービスはどう設計すればいい？」
- 「REST vs GraphQL のトレードオフを分析して」
- 「認証モジュールをリファクタリングしたい。どこから始めればいい？」
- 「このコードベースにアーキテクチャ上の問題はある？」

**skill トリガー例：**

- 「アーキテクチャをレビューして」「ADRを作って」「スケーラビリティを確認して」

---

### code-architect

既存コードベースのパターンを読んだうえで、具体的な実装ブループリント（ファイルパス・インターフェース・ビルド順序）を提供します。

**自動起動トリガー例（agent）：**

- 「通知機能をこのリポジトリに追加したい。どう設計すればいい？」
- 「この機能のブループリントを作って」
- 「実装を始めたいが、何から手をつければいいか分からない」

**skill トリガー例：**

- 「機能を設計して」「実装ブループリントを作って」「ビルド順序を決めて」

---

### code-reviewer

`git diff` またはファイルを読み込み、セキュリティ・品質・パフォーマンスの観点でレビューします。  
指摘は `CRITICAL / HIGH / MEDIUM / LOW` の重大度で分類し、最終判定 `APPROVE / WARNING / BLOCK` を返します。

**自動起動トリガー例（agent）：**

- 「コードをレビューして」
- 「マージ前にチェックして」
- 「SQLインジェクションやXSSの脆弱性がないか確認して」

**skill トリガー例：**

- 「コードをレビューして」「変更に問題がないか確認して」「セキュリティ脆弱性がないか確認して」

---

### code-explorer

LSP（定義ジャンプ・参照検索）を活用して、実行パスや依存関係を深く分析します。  
**ファイルの作成・変更は一切行いません。分析のみ**を出力します。

**自動起動トリガー例（agent）：**

- 「認証フローはどう実装されている？」
- 「このAPIルートが何を呼んでいるか調べて」
- 「新機能を実装する前に既存のデータフローを理解したい」
- 「このクラスがどこで使われているか全部調べて」

**skill トリガー例：**

- 「このコードがどう動いているか調べて」「依存関係をマッピングして」「実行パスを追跡して」

**前提: LSP サーバーのインストール**

LSP 設定はプラグインに同梱済みのため、リポジトリごとの設定は不要です。使いたい言語のサーバーバイナリを PATH に通しておくだけで機能します:

```bash
# TypeScript / JavaScript
npm i -g typescript typescript-language-server

# Python
npm i -g pyright

# PHP
npm i -g intelephense

# Go
go install golang.org/x/tools/gopls@latest  # ~/go/bin を PATH に追加

# Java
brew install jdtls
```

> **TypeScript の注意点**: `typescript-language-server` はワークスペースに `typescript` パッケージ（`node_modules/typescript`）を要求します。`npm install` 前のリポジトリでは LSP が起動しないため、その場合は Grep/Read にフォールバックしてください。

---

### code-simplifier

振る舞いを完全に保ったまま、最近変更されたコードを明確さ・一貫性・保守性の観点で簡潔化します。  
深いネストの解消・デッドコード除去・重複統合・可読性向上を担い、**振る舞いを変える変更は一切行いません**。

**自動起動トリガー例（agent）：**

- 「このコードを簡潔にして」
- 「リファクタして読みやすくして」
- 「最近の変更を整理して」
- 「ネストが深いので浅くして」
- 「デッドコードとconsole.logを除去して」

**skill トリガー例：**

- 「コードを簡潔にして」「重複を整理して」「ネストを浅くして」「デッドコードを除去して」「整理して」

---

### security-reviewer

OWASP Top 10・シークレット検出・インジェクション・SSRF・不適切な暗号などを専門的に検出します。  
ユーザー入力処理・認証・APIエンドポイント・機微データを扱うコードを書いた後にプロアクティブに呼び出してください。  
指摘は `CRITICAL / HIGH / MEDIUM / LOW` の重大度で分類し、最終判定 `APPROVE / WARNING / BLOCK` を返します。**コードの変更は行いません。**

**自動起動トリガー例（agent）：**

- 「セキュリティレビューして」
- 「脆弱性がないか確認して」
- 「認証コードをチェックして」
- 「シークレットが混入していないか確認して」
- 「OWASP Top 10 をチェックして」

**skill トリガー例：**

- 「セキュリティレビューして」「脆弱性スキャンして」「OWASPチェックして」「シークレットが混入していないか確認して」「依存関係に既知の脆弱性がないか確認して」

---

### comment-analyzer

コードコメントの正確性・完全性・長期的価値・誤解を招く要素を分析します。  
**フロントエンドファイル**（HTML/JSX/TSX/Vue/Svelte/CSS等）のコメントはブラウザの検証ツールで誰でも閲覧できるため、**実際に削除**します。非フロントエンドファイルはコードと突合して `Inaccurate / Stale / Incomplete / Low-value` に分類したアドバイザリ指摘を返します。

**自動起動トリガー例（agent）：**

- 「このコメントは正確？」
- 「コメントが古くなっていないか確認して」
- 「ドキュメントコメントをチェックして」
- 「TODO/FIXMEを棚卸しして」
- 「フロントエンドのコメントを消して」

**skill トリガー例：**

- 「コメントをチェックして」「古いコメントを洗い出して」「ドキュメントの過不足を見て」「フロントエンドのコメントを削除して」「コンポーネントのコメントを消して」

---

### network-architect

エンタープライズ・マルチサイトのネットワーク設計を要件から作成します。**設計とレビューに特化しており、設定変更の実行は行いません。**  
詳細な診断・検証・自動化は同梱の5つの専門スキルへ自動的に委譲します。

**自動起動トリガー例（agent）：**

- 「3拠点をつなぐ WAN を設計したい」
- 「データセンター移行のフェーズ計画と検証ゲートを作って」
- 「このキャンパスネットワークのセグメンテーションをレビューして」
- 「IP アドレッシングと VLAN 設計を決めたい」

**同梱スキルとトリガー例：**

| スキル                      | トリガー例                                                                                                                                   |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `network-config-validation` | 「設定を変更前にレビューして」「危険なコマンドがないか確認して」「ACL の参照が壊れていないか調べて」                                         |
| `network-bgp-diagnostics`   | 「BGP ネイバーが Established にならない」「期待しているルートが来ていない」「route-map でフィルタされていないか確認して」                    |
| `network-interface-health`  | 「インターフェースで CRC が出ている」「パケットロスがある」「フラップしているポートを診断して」                                              |
| `cisco-ios-patterns`        | 「安全な show コマンドを教えて」「ワイルドカードマスクの書き方を確認したい」「ACL の向きを確認して」                                         |
| `netmiko-ssh-automation`    | 「Netmiko で show コマンドを複数台から収集したい」「SSH 自動化スクリプトにエラーハンドリングを追加して」「TextFSM パースのパターンを教えて」 |

---

## ディレクトリ構造

```
murosyo-claude-plugin/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   ├── architect/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── agents/architect.md
│   │   └── skills/architect/
│   ├── code-architect/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── agents/code-architect.md
│   │   └── skills/code-architect/
│   ├── code-reviewer/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── agents/code-reviewer.md
│   │   └── skills/code-review/
│   ├── code-explorer/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── agents/code-explorer.md
│   │   └── skills/code-explorer/
│   ├── code-simplifier/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── agents/code-simplifier.md
│   │   └── skills/code-simplifier/
│   ├── security-reviewer/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── agents/security-reviewer.md
│   │   └── skills/security-review/
│   │       ├── SKILL.md
│   │       └── references/
│   ├── comment-analyzer/
│   │   ├── .claude-plugin/plugin.json
│   │   ├── agents/comment-analyzer.md
│   │   └── skills/comment-analyzer/
│   │       ├── SKILL.md
│   │       └── references/patterns.md
│   └── network-architect/
│       ├── .claude-plugin/plugin.json
│       ├── agents/network-architect.md
│       └── skills/
│           ├── network-config-validation/SKILL.md
│           ├── network-bgp-diagnostics/SKILL.md
│           ├── network-interface-health/SKILL.md
│           ├── cisco-ios-patterns/SKILL.md
│           └── netmiko-ssh-automation/SKILL.md
├── README.md
└── LICENSE
```

## ライセンス

[MIT](./LICENSE) © 2026 murosyo01
