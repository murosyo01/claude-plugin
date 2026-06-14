---
name: database-migrations
description: PostgreSQL・MySQL のスキーマ変更・データマイグレーション・ロールバック・ゼロダウンタイムデプロイの安全なパターン集。Prisma・Drizzle・Kysely・Django・TypeORM・golang-migrate に対応。マイグレーションを作成するとき・大テーブルを変更するとき・ゼロダウンタイム戦略を計画するときに参照する。
argument-hint: "[マイグレーションファイルまたはディレクトリ（省略可）]"
allowed-tools: ["Read", "Grep", "Glob", "Bash"]
---

# データベースマイグレーションパターン

本番システムに対する安全・可逆なデータベーススキーマ変更のパターン集。

## いつ有効化するか

- データベーステーブルを作成・変更するとき
- カラムやインデックスを追加・削除するとき
- データマイグレーション（バックフィル・変換）を実行するとき
- ゼロダウンタイムのスキーマ変更を計画するとき
- 新規プロジェクトでマイグレーションツールをセットアップするとき

## 基本原則

1. **すべての変更はマイグレーションで行う** — 本番データベースを直接変更しない
2. **本番ではマイグレーションは前進のみ** — ロールバックは新しい前進マイグレーションで行う
3. **スキーマ変更とデータ変更は分離する** — DDL と DML を一つのマイグレーションに混在させない
4. **本番サイズのデータでテストする** — 100 行で動いても 1000 万行ではロックする可能性がある
5. **マイグレーションは一度デプロイしたら不変** — 本番で実行済みのマイグレーションを編集しない

## マイグレーション安全チェックリスト

マイグレーションを適用する前に確認する:

- [ ] UP と DOWN の両方がある（または明示的に「後退不可」とマークしている）
- [ ] 大テーブルで全テーブルロックが発生しない（並行操作を使用する）
- [ ] 新規カラムにはデフォルト値があるか NULL 許可（NOT NULL をデフォルトなしで追加しない）
- [ ] インデックスは並行作成（既存テーブルへの CREATE TABLE インラインでない）
- [ ] データバックフィルはスキーマ変更と別のマイグレーション
- [ ] 本番データのコピーでテスト済み
- [ ] ロールバック計画が文書化されている

## PostgreSQL パターン

### カラムの安全な追加

```sql
-- 良い例: NULL 許可カラム（ロックなし）
ALTER TABLE users ADD COLUMN avatar_url TEXT;

-- 良い例: デフォルト付きカラム（PostgreSQL 11+ はインスタント、行の書き換えなし）
ALTER TABLE users ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT true;

-- 悪い例: デフォルトなし NOT NULL（既存テーブルの全行書き換えが必要）
ALTER TABLE users ADD COLUMN role TEXT NOT NULL;
-- これはテーブルをロックしてすべての行を書き換える
```

### ダウンタイムなしのインデックス追加

```sql
-- 悪い例: 大テーブルで書き込みをブロックする
CREATE INDEX idx_users_email ON users (email);

-- 良い例: 並行作成、書き込みを許可する
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);

-- 注意: CONCURRENTLY はトランザクションブロック内では実行できない
-- ほとんどのマイグレーションツールは特別な設定が必要
```

### カラムのリネーム（ゼロダウンタイム）

本番環境では直接リネームしない。expand-contract パターンを使用する:

```sql
-- ステップ 1: 新カラムを追加（マイグレーション 001）
ALTER TABLE users ADD COLUMN display_name TEXT;

-- ステップ 2: データをバックフィル（マイグレーション 002、データマイグレーション）
UPDATE users SET display_name = username WHERE display_name IS NULL;

-- ステップ 3: 両方のカラムを読み書きするようにアプリを更新してデプロイ

-- ステップ 4: 旧カラムへの書き込みを停止してから削除（マイグレーション 003）
ALTER TABLE users DROP COLUMN username;
```

### カラムの安全な削除

```sql
-- ステップ 1: アプリのコードからカラム参照をすべて削除
-- ステップ 2: カラム参照なしでアプリをデプロイ
-- ステップ 3: 次のマイグレーションでカラムを削除
ALTER TABLE orders DROP COLUMN legacy_status;
```

### 大規模データマイグレーション

```sql
-- 悪い例: 全行を一つのトランザクションで更新（テーブルをロックする）
UPDATE users SET normalized_email = LOWER(email);

-- 良い例: バッチ更新（進捗表示付き）
DO $$
DECLARE
  batch_size INT := 10000;
  rows_updated INT;
BEGIN
  LOOP
    UPDATE users
    SET normalized_email = LOWER(email)
    WHERE id IN (
      SELECT id FROM users
      WHERE normalized_email IS NULL
      LIMIT batch_size
      FOR UPDATE SKIP LOCKED
    );
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    RAISE NOTICE 'Updated % rows', rows_updated;
    EXIT WHEN rows_updated = 0;
    COMMIT;
  END LOOP;
END $$;
```

## MySQL パターン

### カラムの安全な追加（MySQL 8.0+）

```sql
-- 良い例: INSTANT アルゴリズム（MySQL 8.0.12+、ほぼすべての ADD COLUMN で有効）
ALTER TABLE users ADD COLUMN avatar_url TEXT,
  ALGORITHM=INSTANT;

-- 良い例: NOT NULL + デフォルト（INSTANT で追加可能）
ALTER TABLE users ADD COLUMN is_active TINYINT(1) NOT NULL DEFAULT 1,
  ALGORITHM=INSTANT;

-- 悪い例: NOT NULL をデフォルトなしで追加（全行書き換えが発生）
ALTER TABLE users ADD COLUMN role VARCHAR(20) NOT NULL;
-- MySQL は暗黙的に空文字列をデフォルトにするが、
-- 明示的な DEFAULT がない場合の挙動は SQL モードに依存する
```

### ダウンタイムなしのインデックス追加（MySQL 8.0+）

```sql
-- INPLACE + LOCK=NONE で書き込みをブロックしない
ALTER TABLE large_table ADD INDEX idx_col (col),
  ALGORITHM=INPLACE, LOCK=NONE;

-- 複数インデックスの同時追加（一度の ALTER で効率化）
ALTER TABLE large_table
  ADD INDEX idx_a (col_a),
  ADD INDEX idx_b (col_b),
  ALGORITHM=INPLACE, LOCK=NONE;
```

### 大規模テーブルの変更（gh-ost / pt-online-schema-change）

カラム型変更・テーブルリビルドが必要な操作では、オンラインスキーマ変更ツールを使用する:

```bash
# gh-ost（GitHub 製、複製ベースのオンラインスキーマ変更）
gh-ost \
  --user="$DB_USER" \
  --password="$DB_PASS" \
  --host="$DB_HOST" \
  --database="$DB_NAME" \
  --table="large_table" \
  --alter="MODIFY COLUMN status VARCHAR(50) NOT NULL DEFAULT 'active'" \
  --execute

# pt-online-schema-change（Percona Toolkit）
pt-online-schema-change \
  --alter="MODIFY COLUMN status VARCHAR(50) NOT NULL DEFAULT 'active'" \
  --execute \
  D=$DB_NAME,t=large_table,h=$DB_HOST,u=$DB_USER,p=$DB_PASS
```

### 大規模データマイグレーション（MySQL）

```sql
-- 悪い例: 全行を一つのトランザクションで更新
UPDATE users SET normalized_email = LOWER(email);

-- 良い例: バッチ更新
SET @batch_size = 10000;
SET @last_id = 0;

REPEAT
  UPDATE users
  SET normalized_email = LOWER(email)
  WHERE id > @last_id
    AND normalized_email IS NULL
  ORDER BY id
  LIMIT @batch_size;

  SET @last_id = @last_id + @batch_size;
  -- バッチ間で少し待機してレプリカのラグを軽減する
  DO SLEEP(0.1);
UNTIL ROW_COUNT() = 0 END REPEAT;
```

## Prisma（TypeScript/Node.js）

### ワークフロー

```bash
# スキーマ変更からマイグレーションを作成
npx prisma migrate dev --name add_user_avatar

# 本番環境でペンディングマイグレーションを適用
npx prisma migrate deploy

# データベースのリセット（開発環境のみ）
npx prisma migrate reset

# スキーマ変更後のクライアント生成
npx prisma generate
```

### スキーマ例

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  avatarUrl String?  @map("avatar_url")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  orders    Order[]

  @@map("users")
  @@index([email])
}
```

### カスタム SQL マイグレーション

Prisma が表現できない操作（並行インデックス・データバックフィル）の場合:

```bash
# 空のマイグレーションを作成して SQL を手書きする
npx prisma migrate dev --create-only --name add_email_index
```

```sql
-- migrations/20240115_add_email_index/migration.sql
-- Prisma は CONCURRENTLY を生成できないため手書きする
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users (email);
```

## Drizzle（TypeScript/Node.js）

### ワークフロー

```bash
# スキーマ変更からマイグレーションを生成
npx drizzle-kit generate

# マイグレーションを適用
npx drizzle-kit migrate

# スキーマを直接プッシュ（開発環境のみ、マイグレーションファイルなし）
npx drizzle-kit push
```

### スキーマ例

```typescript
import { pgTable, text, timestamp, uuid, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name"),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});
```

## Kysely（TypeScript/Node.js）

### ワークフロー（kysely-ctl）

```bash
# 設定ファイルを初期化（kysely.config.ts）
kysely init

# 新しいマイグレーションファイルを作成
kysely migrate make add_user_avatar

# ペンディングマイグレーションをすべて適用
kysely migrate latest

# 最後のマイグレーションをロールバック
kysely migrate down

# マイグレーションの状態を確認
kysely migrate list
```

### マイグレーションファイル

```typescript
// migrations/2024_01_15_001_create_user_profile.ts
import { type Kysely, sql } from 'kysely'

// 重要: Kysely<any> を使用（型付き DB インターフェースではない）
// マイグレーションは時間に固定されており、現在のスキーマ型に依存してはならない
export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .createTable('user_profile')
    .addColumn('id', 'serial', (col) => col.primaryKey())
    .addColumn('email', 'varchar(255)', (col) => col.notNull().unique())
    .addColumn('avatar_url', 'text')
    .addColumn('created_at', 'timestamp', (col) =>
      col.defaultTo(sql`now()`).notNull()
    )
    .execute()
}

export async function down(db: Kysely<any>): Promise<void> {
  await db.schema.dropTable('user_profile').execute()
}
```

## Django（Python）

### ワークフロー

```bash
# モデル変更からマイグレーションを生成
python manage.py makemigrations

# マイグレーションを適用
python manage.py migrate

# マイグレーションの状態を確認
python manage.py showmigrations

# カスタム SQL 用の空マイグレーションを生成
python manage.py makemigrations --empty app_name -n description
```

### データマイグレーション

```python
from django.db import migrations

def backfill_display_names(apps, schema_editor):
    User = apps.get_model("accounts", "User")
    batch_size = 5000
    users = User.objects.filter(display_name="")
    while users.exists():
        batch = list(users[:batch_size])
        for user in batch:
            user.display_name = user.username
        User.objects.bulk_update(batch, ["display_name"], batch_size=batch_size)

def reverse_backfill(apps, schema_editor):
    pass  # データマイグレーションは逆戻しなし

class Migration(migrations.Migration):
    dependencies = [("accounts", "0015_add_display_name")]

    operations = [
        migrations.RunPython(backfill_display_names, reverse_backfill),
    ]
```

### SeparateDatabaseAndState

カラムをすぐにドロップせずに Django モデルから削除する:

```python
class Migration(migrations.Migration):
    operations = [
        migrations.SeparateDatabaseAndState(
            state_operations=[
                migrations.RemoveField(model_name="user", name="legacy_field"),
            ],
            database_operations=[],  # まだ DB には触らない
        ),
    ]
```

## golang-migrate（Go）

### ワークフロー

```bash
# マイグレーションペアを作成
migrate create -ext sql -dir migrations -seq add_user_avatar

# ペンディングマイグレーションをすべて適用
migrate -path migrations -database "$DATABASE_URL" up

# 最後のマイグレーションをロールバック
migrate -path migrations -database "$DATABASE_URL" down 1

# バージョンを強制設定（dirty 状態の修復）
migrate -path migrations -database "$DATABASE_URL" force VERSION
```

### マイグレーションファイル

```sql
-- migrations/000003_add_user_avatar.up.sql
ALTER TABLE users ADD COLUMN avatar_url TEXT;
CREATE INDEX CONCURRENTLY idx_users_avatar ON users (avatar_url) WHERE avatar_url IS NOT NULL;

-- migrations/000003_add_user_avatar.down.sql
DROP INDEX IF EXISTS idx_users_avatar;
ALTER TABLE users DROP COLUMN IF EXISTS avatar_url;
```

## ゼロダウンタイムマイグレーション戦略

本番環境の重要な変更には expand-contract パターンを使用する:

```
フェーズ 1: EXPAND（拡張）
  - 新カラム/テーブルを追加（NULL 許可またはデフォルト付き）
  - デプロイ: アプリが旧・新両方に書き込む
  - 既存データをバックフィル

フェーズ 2: MIGRATE（移行）
  - デプロイ: アプリが新から読み込み、両方に書き込む
  - データの整合性を確認

フェーズ 3: CONTRACT（縮小）
  - デプロイ: アプリが新のみ使用
  - 別のマイグレーションで旧カラム/テーブルを削除
```

### タイムライン例

```
Day 1: マイグレーションで new_status カラムを追加（NULL 許可）
Day 1: アプリ v2 をデプロイ — status と new_status の両方に書き込む
Day 2: 既存行のバックフィルマイグレーションを実行
Day 3: アプリ v3 をデプロイ — new_status からのみ読み込む
Day 7: 旧 status カラムを削除するマイグレーションを実行
```

## アンチパターン

| アンチパターン | なぜ問題か | より良いアプローチ |
|-------------|-----------|-----------------|
| 本番環境で手動 SQL を実行 | 監査証跡なし・再現不可 | 常にマイグレーションファイルを使用する |
| デプロイ済みマイグレーションを編集 | 環境間のドリフトを引き起こす | 新しいマイグレーションを作成する |
| デフォルトなしの NOT NULL 追加 | テーブルをロックし全行を書き換える | NULL 許可で追加→バックフィル→制約追加 |
| 大テーブルへのインラインインデックス作成 | 書き込み中にビルドをブロックする | `CREATE INDEX CONCURRENTLY`（PG）/ `ALGORITHM=INPLACE`（MySQL）|
| スキーマとデータを一つのマイグレーションに混在 | ロールバックが困難・長時間トランザクション | マイグレーションを分離する |
| コードを削除する前にカラムを削除 | アプリが存在しないカラムを参照してエラー | コードを先に削除、次のデプロイでカラムを削除 |
| **MySQL のみ**: `ALTER TABLE` に ALGORITHM/LOCK 句を省略 | デフォルトのロック挙動がバージョン・操作によって異なる | 常に `ALGORITHM=INPLACE, LOCK=NONE` を明示する（または gh-ost/pt-osc を使用） |
| **MySQL のみ**: カラム型変更を通常の `ALTER TABLE` で実行 | 大テーブルでフルコピーが発生 | gh-ost または pt-online-schema-change を使用する |

## 関連

- エージェント: `database-reviewer` — データベース全体のレビューワークフロー
- スキル: `postgres-patterns` — PostgreSQL パターン集（Supabase ベストプラクティス）
- スキル: `mysql-patterns` — MySQL/InnoDB パターン集
