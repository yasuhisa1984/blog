---
title: "HTTP完全理解：プロトコルの本質からAPI設計の実務まで"
date: 2025-12-14
draft: false
categories: ["システム設計"]
tags: ["HTTP", "API設計", "REST", "システム設計", "実務"]
description: "HTTPメソッド、ステータスコード、ヘッダーの本質を理解し、実務で使えるAPI設計パターンを身につける。なぜGETで更新してはいけないのか、なぜ適切なステータスコードが重要なのか、理由から理解する。"
cover:
  image: "images/covers/http-api-design.jpg"
  alt: "HTTP API設計"
  relative: false
---

## はじめに

「とりあえずPOSTで送っておけばいい」
「ステータスコードは200か500でいいでしょ」
「ヘッダー？気にしたことない」

こういうAPI、よく見かける。
動くには動く。でも、後から必ず苦労する。

HTTPは「なんとなく」で使えてしまうプロトコルだ。
だからこそ、本質を理解している人と、していない人で差がつく。

この記事では、HTTPの本質から始めて、実務で使えるAPI設計パターンまでを解説する。

---

## HTTPの本質：リクエストとレスポンス

HTTPは、シンプルなプロトコルだ。

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Server as サーバー

    Client->>Server: リクエスト
    Server->>Client: レスポンス
```

これだけ。

### リクエストの構造

```http
POST /api/users HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: Bearer xxxx

{"name": "山田太郎", "email": "yamada@example.com"}
```

| 要素 | 説明 |
|------|------|
| メソッド | POST（何をするか） |
| パス | /api/users（どこに対して） |
| ヘッダー | メタ情報（認証、形式など） |
| ボディ | 実際のデータ |

### レスポンスの構造

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/123

{"id": 123, "name": "山田太郎", "email": "yamada@example.com"}
```

| 要素 | 説明 |
|------|------|
| ステータスコード | 201（結果） |
| ヘッダー | メタ情報（新規リソースの場所など） |
| ボディ | レスポンスデータ |

この構造を理解していれば、どんなAPIでも読める。

---

## HTTPメソッドの本質

### なぜメソッドを使い分けるのか

「全部POSTでいいじゃん」という人がいる。
確かに、技術的には可能だ。

でも、メソッドには**意味**がある。
意味があるから、クライアントもサーバーも適切に振る舞える。

### 各メソッドの意味

```
GET     → 取得する（副作用なし）
POST    → 作成する（新しいリソースを作る）
PUT     → 置換する（リソース全体を入れ替える）
PATCH   → 部分更新する（一部だけ変える）
DELETE  → 削除する
```

### 【重要】安全性と冪等性

この概念を理解していないと、障害を起こす。

| メソッド | 安全 | 冪等 |
|----------|------|------|
| GET | ✅ | ✅ |
| POST | ❌ | ❌ |
| PUT | ❌ | ✅ |
| PATCH | ❌ | ❌ |
| DELETE | ❌ | ✅ |

**安全（Safe）**: サーバーの状態を変えない
**冪等（Idempotent）**: 何度実行しても結果が同じ

#### 実務での意味

```python
# GETは安全 → キャッシュできる、プリフェッチできる
GET /api/users/123

# ブラウザは「このGETリクエスト、先に取得しておこう」ができる
# なぜなら、副作用がないと保証されているから
```

```python
# POSTは冪等じゃない → リトライが危険
POST /api/orders
{"item": "商品A", "quantity": 1}

# タイムアウトしたからリトライ → 2重注文の可能性
# だから、POSTには冪等キーを使う（後述）
```

```python
# PUTは冪等 → リトライしても安全
PUT /api/users/123
{"name": "山田太郎", "email": "yamada@example.com"}

# 何回実行しても、結果は同じ
# リトライしても問題ない
```

---

## 【実務】GETで更新してはいけない理由

たまに見かける、危険なAPI:

```python
# 絶対にやってはいけない
GET /api/users/123/delete
GET /api/orders/456/cancel
```

なぜダメか？

### 1. ブラウザのプリフェッチ

```html
<!-- ブラウザが「このリンク、先に読み込んでおこう」と判断 -->
<a href="/api/users/123/delete">削除</a>

<!-- ユーザーがクリックする前に、削除されてしまう可能性 -->
```

### 2. クローラーのアクセス

```python
# Googlebotが巡回
GET /api/users/123/delete

# 「GETは安全」という前提でアクセス
# でも実際は削除される → データ消失
```

### 3. キャッシュの問題

```python
# CDNやプロキシがキャッシュ
GET /api/users/123/delete

# キャッシュから返される
# 「削除したはず」なのに、されていない
```

**GETは副作用なし。これは絶対のルール。**

---

## ステータスコードの本質

### なぜ適切なステータスコードが重要か

「200で返して、bodyにエラー入れればいい」

```json
// こういうAPI、よく見る
HTTP/1.1 200 OK
{"success": false, "error": "ユーザーが見つかりません"}
```

これの問題:

1. **HTTPクライアントが正常と判断する**
2. **監視ツールがエラーを検知できない**
3. **CDNがエラーレスポンスをキャッシュする**

ステータスコードは、HTTPレイヤーでの「結果の要約」だ。
bodyを見なくても、何が起きたかわかるようにする。

### 主要なステータスコード

#### 2xx: 成功

```http
200 OK              # 成功（一般的）
201 Created         # 作成成功（POSTの成功）
204 No Content      # 成功だがbodyなし（DELETEの成功）
```

#### 3xx: リダイレクト

```http
301 Moved Permanently  # 恒久的な移動（SEOに影響）
302 Found              # 一時的な移動
304 Not Modified       # キャッシュを使え
```

#### 4xx: クライアントエラー

```http
400 Bad Request        # リクエストが不正
401 Unauthorized       # 認証が必要
403 Forbidden          # 認証済みだが権限がない
404 Not Found          # リソースが存在しない
409 Conflict           # 競合（楽観ロックの失敗など）
422 Unprocessable Entity # バリデーションエラー
429 Too Many Requests  # レート制限
```

#### 5xx: サーバーエラー

```http
500 Internal Server Error  # サーバー内部エラー
502 Bad Gateway           # 上流サーバーからの不正なレスポンス
503 Service Unavailable   # 一時的に利用不可
504 Gateway Timeout       # 上流サーバーがタイムアウト
```

### 【実務】ステータスコードの選び方

```python
# ユーザー作成API
def create_user(request):
    # バリデーションエラー → 422
    if not is_valid(request.body):
        return Response(status=422, body={"errors": validation_errors})

    # メールアドレスが既に存在 → 409
    if email_exists(request.body["email"]):
        return Response(status=409, body={"error": "Email already exists"})

    # 作成成功 → 201 + Locationヘッダー
    user = create_user_in_db(request.body)
    return Response(
        status=201,
        headers={"Location": f"/api/users/{user.id}"},
        body=user.to_dict()
    )
```

---

## HTTPヘッダーの実務

### Content-Type: データの形式

```http
Content-Type: application/json          # JSON
Content-Type: application/x-www-form-urlencoded  # フォーム
Content-Type: multipart/form-data       # ファイルアップロード
Content-Type: text/html; charset=utf-8  # HTML
```

**実務での注意**:

```python
# JSONを送るとき、Content-Typeを忘れると...
POST /api/users
{"name": "山田太郎"}

# サーバーが「これはJSONじゃない」と判断する可能性
# 常にContent-Type: application/jsonを付ける
```

### Authorization: 認証

```http
# Bearerトークン（JWT、OAuth）
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Basic認証（Base64エンコード）
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=

# APIキー（カスタムヘッダーの場合も）
X-API-Key: your-api-key-here
```

### Cache-Control: キャッシュ制御

```http
# キャッシュ禁止（個人情報など）
Cache-Control: no-store

# 1時間キャッシュ可能
Cache-Control: max-age=3600

# 共有キャッシュ（CDN）で1日キャッシュ
Cache-Control: public, max-age=86400

# 再検証が必要
Cache-Control: no-cache
```

**実務での使い分け**:

```python
# ユーザー情報 → キャッシュ禁止
GET /api/users/me
Cache-Control: no-store

# 商品一覧 → 短時間キャッシュ
GET /api/products
Cache-Control: max-age=300

# 静的データ（国リストなど） → 長期キャッシュ
GET /api/countries
Cache-Control: max-age=86400
```

### ETag / If-None-Match: 条件付きリクエスト

```http
# 最初のリクエスト
GET /api/users/123
→ 200 OK
→ ETag: "abc123"

# 2回目のリクエスト
GET /api/users/123
If-None-Match: "abc123"
→ 304 Not Modified（変更なし、bodyなし）
```

**帯域節約とパフォーマンス向上に効く。**

---

## REST API設計の実務

### URLの設計原則

```python
# リソースは名詞、複数形
GET  /api/users          # ユーザー一覧
GET  /api/users/123      # 特定のユーザー
POST /api/users          # ユーザー作成
PUT  /api/users/123      # ユーザー更新
DELETE /api/users/123    # ユーザー削除

# ネストしたリソース
GET /api/users/123/orders          # ユーザー123の注文一覧
GET /api/users/123/orders/456      # ユーザー123の注文456
```

```python
# ダメな例
GET /api/getUsers          # 動詞を入れない
GET /api/user              # 複数形にする
POST /api/users/create     # createはメソッドで表現
GET /api/users/123/delete  # GETで削除しない
```

### クエリパラメータの使い方

```python
# フィルタリング
GET /api/users?status=active&role=admin

# ソート
GET /api/users?sort=created_at&order=desc

# ページネーション
GET /api/users?page=2&per_page=20

# 部分取得（フィールド指定）
GET /api/users?fields=id,name,email
```

### ページネーションの実装

#### オフセットベース（シンプルだが大規模で遅い）

```json
GET /api/users?page=2&per_page=20

{
  "data": [...],
  "pagination": {
    "current_page": 2,
    "per_page": 20,
    "total": 1000,
    "total_pages": 50
  }
}
```

```sql
-- 内部的には OFFSET を使う
SELECT * FROM users LIMIT 20 OFFSET 20;

-- 問題: OFFSETが大きくなると遅くなる
SELECT * FROM users LIMIT 20 OFFSET 10000;  -- 遅い！
```

#### カーソルベース（大規模向け）

```json
GET /api/users?cursor=eyJpZCI6MTIzfQ&limit=20

{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_more": true
  }
}
```

```sql
-- 内部的にはWHEREを使う
SELECT * FROM users WHERE id > 123 ORDER BY id LIMIT 20;

-- どれだけデータがあっても高速
```

**実務での選択**:

- 管理画面（データ少ない） → オフセットベース
- 公開API（データ多い） → カーソルベース
- 無限スクロール → カーソルベース一択

---

## 【実務】API設計でよくある問題と解決策

### 問題1: N+1リクエスト

```python
# クライアントが何度もリクエストする羽目に
GET /api/orders
→ [{"id": 1, "user_id": 10}, {"id": 2, "user_id": 20}, ...]

GET /api/users/10  # ユーザー情報を取得
GET /api/users/20  # ユーザー情報を取得
...
```

**解決策: 埋め込み or 一括取得**

```python
# 方法1: 埋め込み（expand）
GET /api/orders?expand=user
→ [{"id": 1, "user": {"id": 10, "name": "山田"}}, ...]

# 方法2: 一括取得エンドポイント
GET /api/users?ids=10,20,30
→ [{"id": 10, ...}, {"id": 20, ...}, ...]
```

### 問題2: 部分更新の表現

```python
# PUTは全体置換
PUT /api/users/123
{"name": "山田", "email": "yamada@example.com", "status": "active"}

# 一部だけ変えたい場合、全フィールド送る必要がある？
```

**解決策: PATCHを使う**

```python
# PATCHは部分更新
PATCH /api/users/123
{"status": "inactive"}  # statusだけ変更
```

### 問題3: 複雑な操作の表現

```python
# 「パスワードリセット」はどう表現する？
# RESTのCRUDに収まらない

# 方法1: サブリソースとして表現
POST /api/users/123/password-reset

# 方法2: アクションとして表現
POST /api/users/123/actions/reset-password

# 方法3: 独立したリソースとして表現
POST /api/password-reset-requests
{"email": "yamada@example.com"}
```

### 問題4: バルク操作

```python
# 100件のユーザーを一括で更新したい
# 100回POSTする？

# 解決策: バルクエンドポイント
PATCH /api/users/bulk
{
  "operations": [
    {"id": 1, "status": "active"},
    {"id": 2, "status": "inactive"},
    ...
  ]
}

# レスポンス
{
  "results": [
    {"id": 1, "success": true},
    {"id": 2, "success": false, "error": "Not found"},
    ...
  ]
}
```

---

## 【実務】エラーレスポンスの設計

### 一貫したエラーフォーマット

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力内容に問題があります",
    "details": [
      {
        "field": "email",
        "message": "メールアドレスの形式が正しくありません"
      },
      {
        "field": "password",
        "message": "パスワードは8文字以上必要です"
      }
    ]
  }
}
```

### エラーコードの設計

```python
# 機械可読なエラーコード
ERROR_CODES = {
    "VALIDATION_ERROR": "入力エラー",
    "AUTHENTICATION_REQUIRED": "認証が必要",
    "PERMISSION_DENIED": "権限がありません",
    "RESOURCE_NOT_FOUND": "リソースが見つかりません",
    "RESOURCE_CONFLICT": "リソースが競合しています",
    "RATE_LIMIT_EXCEEDED": "リクエスト制限を超えました",
    "INTERNAL_ERROR": "内部エラー",
}
```

**クライアントは`code`で分岐、`message`は表示用**:

```javascript
// クライアント側
if (error.code === "RATE_LIMIT_EXCEEDED") {
  // リトライロジック
  await sleep(error.retry_after * 1000);
  return retry();
}
```

---

## 【実務】冪等性の実装

### なぜ冪等性が重要か

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Server as サーバー

    Note over Client: 1回目の注文
    Client->>Server: POST /api/orders
    Server--xClient: タイムアウト<br/>(実際は成功している)

    Note over Client: 「注文できたかわからない<br/>...リトライしよう」
    Client->>Server: POST /api/orders<br/>(同じ内容)
    Server->>Client: 200 OK

    Note over Client,Server: 結果: 2重注文 😱
```

### 冪等キーによる解決

```python
# クライアント側
POST /api/orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000  # UUIDを生成
{"item": "商品A", "quantity": 1}

# サーバー側
def create_order(request):
    idempotency_key = request.headers.get("Idempotency-Key")

    # 既に処理済みか確認
    cached_response = redis.get(f"idempotency:{idempotency_key}")
    if cached_response:
        return cached_response  # 前回と同じレスポンスを返す

    # 新規処理
    order = create_order_in_db(request.body)
    response = {"id": order.id, "status": "created"}

    # レスポンスをキャッシュ（24時間）
    redis.setex(f"idempotency:{idempotency_key}", 86400, response)

    return response
```

**Stripe、Shopifyなど、決済系APIは必ずこの仕組みを持っている。**

---

## 【実務】バージョニング

### なぜバージョニングが必要か

APIは変化する。
でも、既存のクライアントを壊すわけにはいかない。

### バージョニングの方法

#### URLパス方式（最も一般的）

```python
GET /api/v1/users
GET /api/v2/users
```

**メリット**: わかりやすい、キャッシュしやすい
**デメリット**: URLが変わる

#### ヘッダー方式

```python
GET /api/users
Accept: application/vnd.myapp.v2+json
```

**メリット**: URLがきれい
**デメリット**: 実装が複雑、キャッシュしにくい

#### クエリパラメータ方式

```python
GET /api/users?version=2
```

**メリット**: シンプル
**デメリット**: キャッシュキーが複雑になる

**実務ではURLパス方式が多い。迷ったらこれ。**

### 後方互換性を保つ変更

```python
# OK: フィールド追加（後方互換）
# v1: {"id": 1, "name": "山田"}
# v2: {"id": 1, "name": "山田", "email": "yamada@example.com"}

# NG: フィールド削除（後方互換性なし）
# v1: {"id": 1, "name": "山田", "email": "..."}
# v2: {"id": 1, "name": "山田"}  # emailがなくなった！

# NG: フィールド名変更（後方互換性なし）
# v1: {"user_name": "山田"}
# v2: {"name": "山田"}  # フィールド名が変わった！

# NG: 型変更（後方互換性なし）
# v1: {"id": 1}
# v2: {"id": "1"}  # 数値から文字列に変わった！
```

---

## 【実務】認証と認可

### 認証（Authentication）と認可（Authorization）の違い

```
認証: あなたは誰？（ログイン）
認可: あなたは何ができる？（権限チェック）
```

### JWT（JSON Web Token）

```python
# トークン取得
POST /api/auth/login
{"email": "yamada@example.com", "password": "secret"}

→ {"access_token": "eyJhbGciOiJIUzI1NiIs...", "expires_in": 3600}

# APIアクセス
GET /api/users/me
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**JWTの構造**:

```
ヘッダー.ペイロード.署名
eyJhbGciOiJIUzI1NiIs.eyJ1c2VyX2lkIjoxMjM.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

```json
// ペイロード（Base64デコード後）
{
  "user_id": 123,
  "role": "admin",
  "exp": 1702540800  // 有効期限
}
```

### リフレッシュトークン

```python
# アクセストークンは短命（1時間）
# リフレッシュトークンは長命（30日）

# アクセストークン期限切れ時
POST /api/auth/refresh
{"refresh_token": "..."}

→ {"access_token": "新しいトークン", "expires_in": 3600}
```

**なぜ分けるか**:
- アクセストークンが漏洩しても、被害は1時間
- リフレッシュトークンは安全に保管（HttpOnly Cookie）

---

## 【実務】セキュリティ

### CORS（Cross-Origin Resource Sharing）

```python
# サーバー側の設定
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

```python
# Pythonの例（Flask）
from flask_cors import CORS

app = Flask(__name__)
CORS(app, origins=["https://example.com"], supports_credentials=True)
```

### レート制限

```python
# レスポンスヘッダー
X-RateLimit-Limit: 100        # 制限値
X-RateLimit-Remaining: 95     # 残り回数
X-RateLimit-Reset: 1702540800 # リセット時刻（Unix時間）

# 制限超過時
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

### 入力値の検証

```python
# 常にサーバー側でバリデーション
def create_user(request):
    # 型チェック
    if not isinstance(request.body.get("email"), str):
        return error(422, "Email must be a string")

    # 形式チェック
    if not is_valid_email(request.body["email"]):
        return error(422, "Invalid email format")

    # 長さチェック
    if len(request.body.get("name", "")) > 100:
        return error(422, "Name is too long")

    # SQLインジェクション対策（プレースホルダーを使う）
    cursor.execute(
        "INSERT INTO users (name, email) VALUES (?, ?)",
        (request.body["name"], request.body["email"])
    )
```

---

## 実務でのAPI設計チェックリスト

### 設計時

- [ ] URLは名詞・複数形か
- [ ] HTTPメソッドは適切か（GETで更新していないか）
- [ ] ステータスコードは適切か
- [ ] エラーレスポンスは一貫しているか
- [ ] ページネーションは実装されているか
- [ ] バージョニングは考慮されているか

### セキュリティ

- [ ] 認証は適切か（JWT、OAuth）
- [ ] 認可は適切か（権限チェック）
- [ ] CORSは設定されているか
- [ ] レート制限は実装されているか
- [ ] 入力値は検証されているか
- [ ] SQLインジェクション対策はされているか

### パフォーマンス

- [ ] N+1問題は解決されているか
- [ ] キャッシュヘッダーは適切か
- [ ] 大量データはページネーションされているか
- [ ] 不要なフィールドは返していないか

### 運用

- [ ] ログは出力されているか
- [ ] エラーは監視されているか
- [ ] ドキュメントは整備されているか（OpenAPI/Swagger）

---

## まとめ

HTTPとAPI設計の本質は、**意味のある通信**を行うことだ。

- **メソッド**で「何をするか」を伝える
- **ステータスコード**で「結果」を伝える
- **ヘッダー**で「メタ情報」を伝える
- **URL**で「何に対して」を伝える

これらを適切に使えば、
- クライアントは「何が起きたか」を正確に理解できる
- サーバーは「何をすべきか」を正確に理解できる
- 中間のキャッシュやプロキシも適切に動作する

「動けばいい」から「意味のある設計」へ。

それが、プロのAPI設計だ。
