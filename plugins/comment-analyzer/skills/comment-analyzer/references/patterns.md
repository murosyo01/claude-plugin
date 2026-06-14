# コメント分析パターン集

コメント分析における4軸のパターン・検出のヒント・良い例/悪い例・誤検知リファレンス。

---

## 1. Inaccurate（不正確なコメント）

コードと矛盾する説明が含まれているパターン。コメントだけ読むと動作を誤解する。

### 典型パターン

**引数の単位・型が食い違う**

```typescript
// 悪い例: コメントは秒単位と言っているが、実装はミリ秒
/**
 * @param expiresIn - セッション有効期限（秒）
 */
function createSession(userId: string, expiresIn: number) {
  return { userId, expiresAt: Date.now() + expiresIn }; // ms単位で加算している
}

// 良い例: 実装に合わせてミリ秒と明記
/**
 * @param expiresIn - セッション有効期限（ミリ秒）
 */
function createSession(userId: string, expiresIn: number) {
  return { userId, expiresAt: Date.now() + expiresIn };
}
```

**戻り値の説明が実装と食い違う**

```typescript
// 悪い例: コメントはtrueを返すと言っているが、実装はオブジェクトを返す
/**
 * ユーザーを認証する。成功時は true を返す。
 */
function authenticate(token: string): AuthResult {
  // ...
  return { success: true, userId: "abc" };
}

// 良い例: 実際の戻り値型を説明する
/**
 * ユーザーを認証する。成功時は AuthResult オブジェクトを返す。
 */
```

**例外・エラーの説明が実装と食い違う**

```python
# 悪い例: コメントはValueErrorと言っているが、実装はTypeErrorを投げる
def parse_age(value):
    """
    年齢を整数に変換する。
    Raises:
        ValueError: value が整数に変換できない場合
    """
    if not isinstance(value, (int, str)):
        raise TypeError(f"Expected int or str, got {type(value)}")
    return int(value)
```

### 検出のヒント

- 引数の説明と関数シグネチャの型を突合する
- `@returns` / `@return` の説明と実際の `return` 文の型を確認する
- `@throws` / `Raises:` の説明と実際にスローしている例外クラスを確認する
- 「〇〇を返す」という説明が複数の実装パスで一貫しているかを確認する

---

## 2. Stale（腐敗した古いコメント）

以前は正確だったが、コードの変更後に更新されなかったコメント。

### 典型パターン

**削除した引数が説明に残っている**

```typescript
// 悪い例: timeout 引数は削除されたが説明が残っている
/**
 * データベースに接続する。
 * @param host - 接続先ホスト
 * @param port - ポート番号
 * @param timeout - タイムアウト秒数（廃止済み）  ← まだ残っている
 */
function connect(host: string, port: number) { // timeout は削除済み
  // ...
}

// 良い例: 削除した引数の説明も合わせて除去する
/**
 * データベースに接続する。
 * @param host - 接続先ホスト
 * @param port - ポート番号
 */
```

**存在しないモジュール・クラスへの参照**

```python
# 悪い例: LegacyUserService は削除済み
# UserService の代わりに LegacyUserService を使うこともできる
class UserService:
    pass
```

**古い実装方針が説明として残っている**

```typescript
// 悪い例: キャッシュを廃止したのに説明だけ残っている
/**
 * ユーザー情報を取得する。結果はRedisにキャッシュされる。
 */
async function getUser(id: string): Promise<User> {
  // Redisキャッシュは廃止。直接DBから取得する
  return db.users.findById(id);
}
```

**コメントのバージョン番号・日付が古いまま**

```java
// 悪い例: バージョン番号が更新されていない
/**
 * v1.2 で追加。v2.0 で非推奨。
 * 現在のバージョン: v3.5
 */
```

### 検出のヒント

- `@param` の引数名と実際の関数シグネチャを突合する
- コメント内のクラス名・関数名・モジュール名を `Grep` で検索し、実際に存在するか確認する
- 「〜は〜を使う」「〜に保存される」のような外部依存の説明を実装と突合する

---

## 3. Incomplete（不完全なコメント）

説明が存在するが、重要な情報が欠落しているパターン。特に公開APIで問題になる。

### 典型パターン

**副作用が記述されていない**

```typescript
// 悪い例: キャッシュを更新する副作用が記述されていない
/**
 * ユーザー情報を更新する。
 */
async function updateUser(id: string, data: Partial<User>): Promise<User> {
  const user = await db.users.update(id, data);
  cache.invalidate(`user:${id}`); // 副作用: キャッシュも消す
  return user;
}

// 良い例: 副作用を明記する
/**
 * ユーザー情報を更新する。更新後、対応するキャッシュエントリを無効化する。
 */
```

**前提条件・制約が記述されていない**

```python
# 悪い例: 引数の制約が記述されていない
def divide(a: float, b: float) -> float:
    """2つの数を割り算する。"""
    return a / b  # b=0 の場合は ZeroDivisionError

# 良い例: 制約と例外を明記する
def divide(a: float, b: float) -> float:
    """
    2つの数を割り算する。
    Args:
        a: 被除数
        b: 除数（0以外を指定すること）
    Returns:
        a / b の結果
    Raises:
        ZeroDivisionError: b が 0 の場合
    """
    return a / b
```

**非自明なアルゴリズムに説明がない**

```typescript
// 悪い例: なぜこの実装なのかが分からない
function throttle<T extends (...args: unknown[]) => void>(fn: T, delay: number): T {
  let lastCall = 0;
  return ((...args) => {
    const now = Date.now();
    if (now - lastCall >= delay) {
      lastCall = now;
      fn(...args);
    }
  }) as T;
}

// 良い例: 設計意図を説明する
/**
 * 関数の呼び出し頻度を制限するスロットル実装。
 * タイマーリセット方式ではなく「最終呼び出し時刻との差分」方式を使うため、
 * 連続入力の先頭と末尾両方で fn が呼ばれることはない（先頭のみ）。
 */
```

### 検出のヒント

- エクスポートされた関数・クラス・型に注目する（内部ヘルパーは対象外）
- `throw` / `raise` している例外がドキュメント化されているかを確認する
- 外部APIを呼ぶ、DBを書き込む、キャッシュを操作するなどの副作用が説明されているかを確認する
- 複雑な条件分岐やアルゴリズム（O(n²)以上・非直感的な実装）に説明があるかを確認する

---

## 4. Low-value（価値が低いコメント）

あっても読者に何も付加価値を与えないコメント。削除することで可読性が上がるケース。

### 典型パターン

**コードをそのまま言い換えているだけ**

```typescript
// 悪い例: 変数名を読めば自明
// ユーザーIDを取得する
const userId = getUserId();

// カウントをインクリメントする
count++;

// 配列が空かどうか確認する
if (array.length === 0) { ... }
```

```python
# 悪い例: ループの説明がコードそのまま
# リスト内の各アイテムをイテレートする
for item in items:
    process(item)
```

**自明な関数への説明**

```typescript
// 悪い例: 関数名で十分
/** ユーザー名を返す */
function getUserName(user: User): string {
  return user.name;
}

// 良い例: コメントなし（または関数名をより説明的にする）
function getUserName(user: User): string {
  return user.name;
}
```

**型シグネチャの言い換え**

```typescript
// 悪い例: TypeScriptの型定義で自明
/**
 * @param id - string型のユーザーID
 * @returns Promise<User>型のユーザーオブジェクト
 */
async function getUser(id: string): Promise<User> { ... }
```

### 検出のヒント

- 「〇〇する」で終わるコメントで、その〇〇が関数名・変数名と同義のものを探す
- コードを読まずにコメントだけで説明できる「情報の上乗せ」がないか確認する
- Low-valueとして指摘する際は削除を提案するが、**削除を実行しない**（分析専用）

---

## スキップすべき誤検知

以下のパターンはフラグを立てない:

| パターン | 理由 |
|---------|------|
| ライセンスヘッダー | 法的要件のある定型コメント |
| `// eslint-disable-next-line` 等のツール指示コメント | ツールへの指示であり読者向けの説明ではない |
| 生成コードのコメント（`// Code generated by ...`） | 手動での更新を想定していない |
| セクション区切り（`// ===== ...`） | 大きなファイルのナビゲーション補助 |
| 自明な内部ヘルパーのコメント欠落 | `const double = (n: number) => n * 2` のようなケース |
| TODO に対する「チケット番号があるべき」| チームの規約が不明な状態での強制 |
| 「英語で書くべき」| プロジェクト全体が日本語コメントを使っている場合 |
