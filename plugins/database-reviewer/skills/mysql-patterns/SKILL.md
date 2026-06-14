---
name: mysql-patterns
description: MySQL（InnoDB）のクエリ最適化・スキーマ設計・インデックス・権限・接続設定に関するパターン集。SQL を書くとき・スキーマを設計するとき・スロークエリをトラブルシュートするとき・テーブル変更を計画するときに参照する。
argument-hint: "[対象のクエリ・スキーマ・ファイル（省略可）]"
allowed-tools: ["Read", "Grep", "Glob", "Bash"]
---

# MySQL パターン

MySQL 8.0（InnoDB）のベストプラクティスクイックリファレンス。詳細なレビューが必要な場合は `database-reviewer` エージェントを使用する。

## いつ有効化するか

- SQL クエリやマイグレーションを書くとき
- データベーススキーマを設計するとき
- スロークエリをトラブルシュートするとき
- テーブルの ALTER を計画するとき
- ユーザー権限・コネクション設定を変更するとき

## クイックリファレンス

### インデックスチートシート

| クエリパターン | インデックス種別 | 例 |
|--------------|----------------|---|
| `WHERE col = value` | B-tree（デフォルト） | `CREATE INDEX idx ON t (col)` |
| `WHERE col > value` | B-tree | `CREATE INDEX idx ON t (col)` |
| `WHERE a = x AND b > y` | 複合 | `CREATE INDEX idx ON t (a, b)` |
| `MATCH(col) AGAINST(...)` | FULLTEXT | `CREATE FULLTEXT INDEX idx ON t (col)` |
| `WHERE col LIKE 'prefix%'` | プレフィックス | `CREATE INDEX idx ON t (col(20))` |
| JSON フィールド検索 | 関数インデックス | `CREATE INDEX idx ON t ((col->>'$.key'))` |

### データ型クイックリファレンス

| ユースケース | 正しい型 | 避ける型 |
|------------|---------|---------|
| ID | `BIGINT UNSIGNED AUTO_INCREMENT` | `INT`・ランダム UUID |
| 文字列（短い） | `VARCHAR(191)` | `TEXT`（インデックス制限あり） |
| 文字列（長い） | `TEXT` | `VARCHAR(65535)` |
| タイムスタンプ | `DATETIME` または `TIMESTAMP` ※ | `INT`（Unix 時間） |
| 金額 | `DECIMAL(10,2)` | `FLOAT`・`DOUBLE` |
| フラグ | `TINYINT(1)` または `BOOLEAN` | `VARCHAR`・`CHAR` |
| UUID（文字列） | `CHAR(36)` | `VARCHAR(255)` |

※ `TIMESTAMP` は 2038 年問題があるため `DATETIME` を推奨（アプリ側でタイムゾーンを管理する場合）

### よく使うパターン

**複合インデックスの順序（等値カラムを先に）:**
```sql
-- status で絞り込んでから created_at で範囲指定するクエリ向け
CREATE INDEX idx ON orders (status, created_at);
-- WHERE status = 'pending' AND created_at > '2024-01-01' で有効
```

**カバリングインデックス（テーブルルックアップを排除）:**
```sql
-- SELECT email, name, created_at WHERE email = ? でインデックスのみで解決
CREATE INDEX idx ON users (email, name, created_at);
-- MySQL では INCLUDE 句が使えないため、SELECT するカラムも含める
```

**部分インデックス相当（ソフトデリート対応）:**
```sql
-- MySQL 8.0.13+: 関数インデックスで部分インデックスを模倣
CREATE INDEX idx ON users ((CASE WHEN deleted_at IS NULL THEN email END));

-- または生成カラムを使用
ALTER TABLE users ADD COLUMN active_email VARCHAR(255)
  GENERATED ALWAYS AS (IF(deleted_at IS NULL, email, NULL)) VIRTUAL;
CREATE INDEX idx ON users (active_email);
```

**UPSERT（INSERT ... ON DUPLICATE KEY UPDATE）:**
```sql
INSERT INTO settings (user_id, `key`, value)
VALUES (123, 'theme', 'dark')
ON DUPLICATE KEY UPDATE value = VALUES(value);

-- MySQL 8.0.20+ では VALUES() の代わりに aliases を推奨
INSERT INTO settings (user_id, `key`, value) AS new
VALUES (123, 'theme', 'dark')
ON DUPLICATE KEY UPDATE value = new.value;
```

**カーソルページネーション（OFFSET の代替）:**
```sql
-- O(1) vs OFFSET の O(n)
SELECT * FROM products WHERE id > ? ORDER BY id LIMIT 20;
```

**キュー処理（SKIP LOCKED、MySQL 8.0+）:**
```sql
-- 競合なしでワーカーが次のジョブを取得する
SELECT id FROM jobs WHERE status = 'pending'
ORDER BY created_at LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE jobs SET status = 'processing' WHERE id = ?;
```

**デッドロック防止（一貫したロック順序）:**
```sql
-- 複数行を更新するときは常に id 順でロックを取得する
SELECT id FROM orders WHERE user_id = ? ORDER BY id FOR UPDATE;
```

### アンチパターン検出クエリ

```sql
-- 未使用インデックスの確認（sys スキーマ）
SELECT object_schema, object_name, index_name
FROM sys.schema_unused_indexes
WHERE object_schema = 'your_db';

-- スロークエリの確認（performance_schema）
SELECT digest_text, avg_timer_wait/1000000000 AS avg_ms, count_star AS calls
FROM performance_schema.events_statements_summary_by_digest
WHERE schema_name = 'your_db'
ORDER BY avg_timer_wait DESC LIMIT 10;

-- テーブルブロートの確認
SELECT table_name,
  ROUND(data_length/1024/1024, 2) AS data_mb,
  ROUND(data_free/1024/1024, 2) AS free_mb
FROM information_schema.tables
WHERE table_schema = 'your_db' AND data_free > 1024*1024*10
ORDER BY data_free DESC;

-- 外部キーのインデックス確認
-- InnoDB は FK 作成時に自動インデックスを作成するが、
-- 既存カラムへの FK 追加後は明示的に確認する
SELECT kcu.constraint_name, kcu.column_name, kcu.referenced_table_name
FROM information_schema.key_column_usage kcu
LEFT JOIN information_schema.statistics s
  ON s.table_schema = kcu.table_schema
  AND s.table_name = kcu.table_name
  AND s.column_name = kcu.column_name
WHERE kcu.referenced_table_name IS NOT NULL
  AND kcu.table_schema = 'your_db'
  AND s.index_name IS NULL;
```

### オンライン DDL（大テーブルの ALTER）

```sql
-- インデックス追加（ロックなし、MySQL 8.0+）
ALTER TABLE large_table
  ADD INDEX idx_col (col),
  ALGORITHM=INPLACE, LOCK=NONE;

-- カラム追加（instant、MySQL 8.0.12+）
ALTER TABLE large_table
  ADD COLUMN new_col TEXT,
  ALGORITHM=INSTANT;

-- NOT NULL カラムの追加（デフォルト値を必ず指定）
-- デフォルトなしは全行書き換えを引き起こす
ALTER TABLE large_table
  ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active',
  ALGORITHM=INSTANT;

-- カラム型変更やリビルドが必要な操作には gh-ost または pt-online-schema-change を使用
-- gh-ost: https://github.com/github/gh-ost
-- pt-osc: https://www.percona.com/software/database-tools/percona-toolkit
```

### 設定テンプレート

```sql
-- InnoDB バッファプール（RAM の 70-80% が目安）
SET GLOBAL innodb_buffer_pool_size = 1073741824; -- 1GB

-- コネクション設定
SET GLOBAL max_connections = 200;
SET GLOBAL wait_timeout = 28800;       -- 8 時間（接続アイドルタイムアウト）
SET GLOBAL interactive_timeout = 28800;

-- スロークエリログの有効化
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;        -- 1 秒以上のクエリをログ
SET GLOBAL log_queries_not_using_indexes = 1;

-- 文字セット（utf8mb4 を必ず使用）
SET GLOBAL character_set_server = 'utf8mb4';
SET GLOBAL collation_server = 'utf8mb4_unicode_ci';

-- タイムゾーン（明示的に設定する）
SET GLOBAL time_zone = '+00:00';
```

### 権限管理

```sql
-- アプリケーションユーザーには必要最小限の権限のみ付与する
-- GRANT ALL は使用しない
CREATE USER 'app_user'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'app_user'@'%';

-- 読み取り専用レプリカ用ユーザー
CREATE USER 'readonly_user'@'%' IDENTIFIED BY 'strong_password';
GRANT SELECT ON myapp.* TO 'readonly_user'@'%';

-- マイグレーション用ユーザー（スキーマ変更権限を持つ）
CREATE USER 'migration_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALTER, CREATE, DROP, INDEX ON myapp.* TO 'migration_user'@'localhost';
```

## 関連

- エージェント: `database-reviewer` — データベース全体のレビューワークフロー
- スキル: `postgres-patterns` — PostgreSQL 版のパターン集（Supabase ベストプラクティス）
- スキル: `database-migrations` — 安全なマイグレーションパターン（PG/MySQL/ORM 対応）
