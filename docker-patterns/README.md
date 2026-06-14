# docker-patterns

Docker および Docker Compose のベストプラクティスを Claude Code に提供するプラグイン。

## 概要

ローカル開発、コンテナセキュリティ、ネットワーキング、ボリューム戦略、マルチサービスオーケストレーションに関するパターン集をスキルとして提供します。

## スキル一覧

| スキル | 内容 |
|--------|------|
| `docker-patterns` | Docker / Docker Compose のベストプラクティス全般 |

## 自動発動トリガー

以下のような作業をしているときに Claude が自動でスキルを参照します：

- ローカル開発用 Docker Compose のセットアップ
- マルチコンテナ構成の設計
- コンテナネットワークやボリュームのトラブルシューティング
- Dockerfile のセキュリティ・サイズレビュー
- ローカル開発からコンテナ化環境への移行

## カバーする内容

- **Docker Compose**: Web アプリスタックの標準構成、override ファイル
- **マルチステージ Dockerfile**: dev / build / production の分離
- **ネットワーキング**: サービスディスカバリー、カスタムネットワーク
- **ボリューム戦略**: named volume、bind mount、anonymous volume
- **コンテナセキュリティ**: 非 root 実行、capability 削除、シークレット管理
- **デバッグ**: ログ確認、シェル接続、ネットワーク診断コマンド
- **アンチパターン**: よくある間違いと回避方法

## インストール

```bash
# ローカルテスト
cc --plugin-dir /path/to/docker-patterns

# プロジェクトへのコピー
cp -r docker-patterns /your-project/.claude-plugin-local/
```
