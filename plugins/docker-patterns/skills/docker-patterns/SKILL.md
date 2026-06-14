---
name: docker-patterns
description: このスキルは、ユーザーが「Docker Compose を設定したい」「Dockerfile を書きたい」「コンテナのネットワークを設定したい」「ボリュームを管理したい」「コンテナのセキュリティを強化したい」「ローカル開発環境をコンテナ化したい」「マルチコンテナ構成を設計したい」「docker compose up が動かない」「コンテナ間の通信ができない」と依頼したときに使用する。ローカル開発から本番環境まで、Docker / Docker Compose のベストプラクティスとパターンを提供する。
argument-hint: "[対象サービス・スタック・問題の概要（省略可）]"
allowed-tools: ["Read", "Write", "Edit", "Bash"]
---

# Docker パターン

Docker および Docker Compose のベストプラクティスとパターン集。

## Docker Compose — ローカル開発標準構成

### Web アプリスタック

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: dev                     # マルチステージ Dockerfile の dev ステージを使用
    ports:
      - "3000:3000"
    volumes:
      - .:/app                        # ホットリロード用バインドマウント
      - /app/node_modules             # コンテナの依存関係を保護する匿名ボリューム
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db:5432/app_dev
      - REDIS_URL=redis://redis:6379/0
      - NODE_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: npm run dev

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

  mailpit:                            # ローカルメールテスト
    image: axllent/mailpit
    ports:
      - "8025:8025"                   # Web UI
      - "1025:1025"                   # SMTP

volumes:
  pgdata:
  redisdata:
```

### マルチステージ Dockerfile（開発・本番分離）

```dockerfile
# ステージ: 依存関係
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# ステージ: dev（ホットリロード・デバッグツール込み）
FROM node:22-alpine AS dev
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# ステージ: build
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --production

# ステージ: production（最小イメージ）
FROM node:22-alpine AS production
WORKDIR /app
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001
USER appuser
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./
ENV NODE_ENV=production
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### override ファイル

```yaml
# docker-compose.override.yml（自動読み込み、開発環境専用）
services:
  app:
    environment:
      - DEBUG=app:*
      - LOG_LEVEL=debug
    ports:
      - "9229:9229"                   # Node.js デバッガー

# docker-compose.prod.yml（本番環境用、明示的に指定）
services:
  app:
    build:
      target: production
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```

```bash
# 開発（override が自動ロードされる）
docker compose up

# 本番
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## ネットワーキング

### サービスディスカバリー

同じ Compose ネットワーク内のサービスはサービス名で名前解決できる：

```
# "app" コンテナから:
postgres://postgres:postgres@db:5432/app_dev    # "db" が db コンテナに解決される
redis://redis:6379/0                             # "redis" が redis コンテナに解決される
```

### カスタムネットワーク

```yaml
services:
  frontend:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net
      - backend-net

  db:
    networks:
      - backend-net              # api からのみ到達可能、frontend からは不可

networks:
  frontend-net:
  backend-net:
```

### 公開ポートの最小化

```yaml
services:
  db:
    ports:
      - "127.0.0.1:5432:5432"   # ホストのみからアクセス可能
    # 本番環境ではポートを省略 — Docker ネットワーク内のみでアクセス
```

## ボリューム戦略

```yaml
volumes:
  # Named volume: コンテナ再起動後も永続化、Docker が管理
  pgdata:

  # Bind mount: ホストディレクトリをコンテナにマップ（開発用）
  # - ./src:/app/src

  # Anonymous volume: バインドマウントが上書きするコンテナ生成コンテンツを保護
  # - /app/node_modules
```

### よく使うパターン

```yaml
services:
  app:
    volumes:
      - .:/app                   # ソースコード（ホットリロード用バインドマウント）
      - /app/node_modules        # コンテナの node_modules をホストから保護
      - /app/.next               # ビルドキャッシュを保護

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data                              # 永続データ
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql     # 初期化スクリプト
```

## コンテナセキュリティ

### Dockerfile ハードニング

```dockerfile
# 1. 特定タグを使用（:latest は禁止）
FROM node:22.12-alpine3.20

# 2. 非 root で実行
RUN addgroup -g 1001 -S app && adduser -S app -u 1001
USER app

# 3. capability の削除（Compose 側で設定）
# 4. 可能な限りルートファイルシステムを読み取り専用にする
# 5. イメージレイヤーにシークレットを含めない
```

### Compose セキュリティ設定

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /app/.cache
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE          # 1024 未満のポートにバインドする場合のみ
```

### シークレット管理

```yaml
# 推奨: 実行時に環境変数を注入
services:
  app:
    env_file:
      - .env                     # git にコミットしない
    environment:
      - API_KEY                  # ホスト環境から継承

# 推奨: Docker secrets（Swarm モード）
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    secrets:
      - db_password

# 禁止: イメージにハードコード
# ENV API_KEY=sk-proj-xxxxx      # NEVER DO THIS
```

## .dockerignore

```
node_modules
.git
.env
.env.*
dist
coverage
*.log
.next
.cache
docker-compose*.yml
Dockerfile*
README.md
tests/
```

## デバッグ

### よく使うコマンド

```bash
# ログ確認
docker compose logs -f app           # app のログを追跡
docker compose logs --tail=50 db     # db の最新 50 行

# 実行中コンテナでコマンドを実行
docker compose exec app sh           # app にシェルで入る
docker compose exec db psql -U postgres  # postgres に接続

# 状態確認
docker compose ps                     # 実行中サービス一覧
docker compose top                    # 各コンテナのプロセス
docker stats                          # リソース使用量

# 再ビルド
docker compose up --build             # イメージを再ビルド
docker compose build --no-cache app   # 強制フルリビルド

# クリーンアップ
docker compose down                   # コンテナを停止・削除
docker compose down -v                # ボリュームも削除（破壊的）
docker system prune                   # 未使用イメージ・コンテナを削除
```

### ネットワーク問題のデバッグ

```bash
# コンテナ内で DNS 解決を確認
docker compose exec app nslookup db

# 疎通確認
docker compose exec app wget -qO- http://api:3000/health

# ネットワーク確認
docker network ls
docker network inspect <project>_default
```

## アンチパターン

```
# BAD: オーケストレーションなしで本番環境に Docker Compose を使う
# 本番のマルチコンテナは Kubernetes / ECS / Docker Swarm を使う

# BAD: ボリュームなしでコンテナにデータを保存する
# コンテナは揮発的 — ボリュームなしで再起動するとデータが消える

# BAD: root で実行する
# 必ず非 root ユーザーを作成して使う

# BAD: :latest タグを使う
# 再現可能なビルドのために特定バージョンにピン留めする

# BAD: 全サービスを 1 つの大きなコンテナにまとめる
# 関心を分離する — 1 プロセス 1 コンテナ

# BAD: docker-compose.yml にシークレットを書く
# .env ファイル（gitignore 済み）または Docker secrets を使う
```
