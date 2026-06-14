---
name: postgres-patterns
description: PostgreSQL のクエリ最適化・スキーマ設計・インデックス・RLS・接続設定に関するパターン集。Supabase ベストプラクティスに基づく。SQL を書くとき・スキーマを設計するとき・スロークエリをトラブルシュートするとき・Row Level Security を実装するときに参照する。
argument-hint: "[対象のクエリ・スキーマ・ファイル（省略可）]"
allowed-tools: ["Read", "Grep", "Glob", "Bash"]
---

# PostgreSQL パターン

PostgreSQL のベストプラクティスクイックリファレンス。詳細なレビューが必要な場合は `database-reviewer` エージェントを使用する。

## いつ有効化するか

- SQL クエリやマイグレーションを書くとき
- データベーススキーマを設計するとき
- スロークエリをトラブルシュートするとき
- Row Level Security を実装するとき
- コネクションプーリングを設定するとき

## クイックリファレンス

### インデックスチートシート

| クエリパターン | インデックス種別 | 例 |
|--------------|----------------|---|
| `WHERE col = value` | B-tree（デフォルト） | `CREATE INDEX idx ON t (col)` |
| `WHERE col > value` | B-tree | `CREATE INDEX idx ON t (col)` |
| `WHERE a = x AND b > y` | 複合 | `CREATE INDEX idx ON t (a, b)` |
| `WHERE jsonb @> '{}'` | GIN | `CREATE INDEX idx ON t USING gin (col)` |
| `WHERE tsv @@ query` | GIN | `CREATE INDEX idx ON t USING gin (col)` |
| 時系列範囲検索 | BRIN | `CREATE INDEX idx ON t USING brin (col)` |

### データ型クイックリファレンス

| ユースケース | 正しい型 | 避ける型 |
|------------|---------|---------|
| ID | `bigint` | `int`・ランダム UUID |
| 文字列 | `text` | `varchar(255)` |
| タイムスタンプ | `timestamptz` | `timestamp` |
| 金額 | `numeric(10,2)` | `float` |
| フラグ | `boolean` | `varchar`・`int` |

### よく使うパターン

**複合インデックスの順序（等値カラムを先に）:**
```sql
-- status で絞り込んでから created_at で範囲指定するクエリ向け
CREATE INDEX idx ON orders (status, created_at);
-- WHERE status = 'pending' AND created_at > '2024-01-01' で有効
```

**カバリングインデックス（テーブルルックアップを排除）:**
```sql
CREATE INDEX idx ON users (email) INCLUDE (name, created_at);
-- SELECT email, name, created_at FROM users WHERE email = ? でテーブル参照不要
```

**部分インデックス（ソフトデリート対応）:**
```sql
CREATE INDEX idx ON users (email) WHERE deleted_at IS NULL;
-- アクティブユーザーのみ対象の小さなインデックス
```

**RLS ポリシー（最適化済み）:**
```sql
-- 正しい例: (SELECT auth.uid()) で行ごとの関数呼び出しを防ぐ
CREATE POLICY policy ON orders
  USING ((SELECT auth.uid()) = user_id);

-- 誤った例: 行ごとに auth.uid() が評価される（インデックスも効かない）
CREATE POLICY policy ON orders
  USING (auth.uid() = user_id);
```

**UPSERT:**
```sql
INSERT INTO settings (user_id, key, value)
VALUES (123, 'theme', 'dark')
ON CONFLICT (user_id, key)
DO UPDATE SET value = EXCLUDED.value;
```

**カーソルページネーション（OFFSET の代替）:**
```sql
-- O(1) vs OFFSET の O(n)
SELECT * FROM products WHERE id > $last_id ORDER BY id LIMIT 20;
```

**キュー処理（SKIP LOCKED）:**
```sql
-- 競合なしでワーカーが次のジョブを取得する
UPDATE jobs SET status = 'processing'
WHERE id = (
  SELECT id FROM jobs WHERE status = 'pending'
  ORDER BY created_at LIMIT 1
  FOR UPDATE SKIP LOCKED
) RETURNING *;
```

### アンチパターン検出クエリ

```sql
-- 未インデックス外部キーの検出
SELECT conrelid::regclass, a.attname
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    WHERE i.indrelid = c.conrelid AND a.attnum = ANY(i.indkey)
  );

-- スロークエリの確認（pg_stat_statements 要有効化）
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC;

-- テーブルブロートの確認
SELECT relname, n_dead_tup, last_vacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### 設定テンプレート

```sql
-- コネクション制限（RAM に合わせて調整）
ALTER SYSTEM SET max_connections = 100;
ALTER SYSTEM SET work_mem = '8MB';

-- タイムアウト
ALTER SYSTEM SET idle_in_transaction_session_timeout = '30s';
ALTER SYSTEM SET statement_timeout = '30s';

-- モニタリング
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- セキュリティデフォルト
REVOKE ALL ON SCHEMA public FROM public;

SELECT pg_reload_conf();
```

## 関連

- エージェント: `database-reviewer` — データベース全体のレビューワークフロー
- スキル: `mysql-patterns` — MySQL/InnoDB 版のパターン集
- スキル: `database-migrations` — 安全なマイグレーションパターン（PG/MySQL/ORM 対応）

---

*Supabase Agent Skills（credit: Supabase team）のパターンを基に作成（MIT License）*
