---
title: "Rate Limiting完全理解：アルゴリズムから実装、分散環境での運用まで"
date: 2025-12-14
draft: false
categories: ["システム設計"]
tags: ["Rate Limiting", "API", "セキュリティ", "システム設計", "Redis"]
description: "なぜRate Limitingが必要か、各アルゴリズムの特徴と使い分け、Redis/Nginxでの実装、分散環境での課題と解決策、429レスポンスの正しい返し方まで実務で必要な知識を網羅。"
cover:
  image: "images/covers/rate-limiting.jpg"
  alt: "Rate Limiting"
  relative: false
---

## はじめに

「APIが落ちた。原因は1人のユーザーが秒間1000リクエスト投げてきたから」

これ、笑い話じゃない。実際に起きる。

悪意がなくても起きる。
- バグで無限ループになったクライアント
- リトライを連打する実装ミス
- スクレイピングボット

**Rate Limitingは、システムを守る最後の防衛線だ。**

この記事では、Rate Limitingの本質から、アルゴリズム、実装、分散環境での運用までを解説する。

---

## なぜRate Limitingが必要か

### 守るべきもの

| 対象 | 脅威 |
|------|------|
| サーバーリソース | 過負荷によるダウン |
| データベース | 大量クエリによる性能低下 |
| 外部API | 課金上限、利用規約違反 |
| 他のユーザー | 一部ユーザーによるリソース独占 |

### Rate Limitingの効果

```
Before (Rate Limiting なし):
User A: 1000 req/s → サーバー処理 → DB過負荷 → 全員に影響

After (Rate Limiting あり):
User A: 1000 req/s → Rate Limiter (10 req/s まで) → サーバー処理
        ↓ 990 req/s は 429 Too Many Requests
```

---

## Rate Limitingアルゴリズム

### 1. 固定ウィンドウ（Fixed Window）

```
Window: 1分

0:00 - 0:59  → 100リクエストまで
1:00 - 1:59  → 100リクエストまで
2:00 - 2:59  → 100リクエストまで

|-------- Window 1 --------|-------- Window 2 --------|
0:00                    1:00                      2:00
|  ←  100 req まで OK  →  |  ←  100 req まで OK  →  |
```

**実装**:

```python
import time

class FixedWindowRateLimiter:
    def __init__(self, limit, window_size):
        self.limit = limit          # 100
        self.window_size = window_size  # 60秒
        self.counters = {}  # {user_id: (window_start, count)}

    def is_allowed(self, user_id):
        now = time.time()
        window_start = int(now // self.window_size) * self.window_size

        if user_id not in self.counters:
            self.counters[user_id] = (window_start, 0)

        stored_window, count = self.counters[user_id]

        # 新しいウィンドウなら、カウントをリセット
        if stored_window != window_start:
            self.counters[user_id] = (window_start, 1)
            return True

        # 制限内なら許可
        if count < self.limit:
            self.counters[user_id] = (window_start, count + 1)
            return True

        return False
```

**メリット**:
- 実装がシンプル
- メモリ効率が良い

**デメリット**:
- **バースト問題**: ウィンドウの境界で2倍のリクエストが通る

```
Window境界でのバースト:
|-------- Window 1 --------|-------- Window 2 --------|
                      0:59 | 1:00
                100 req →  | ← 100 req
                           ↑
                   この1秒間に200リクエストが通る
```

### 2. スライディングウィンドウログ（Sliding Window Log）

```
現在時刻: 1:30

過去1分間のリクエストをカウント
[1:00, 1:05, 1:10, 1:15, 1:20, 1:25, 1:29]
↑ 0:30 より前は除外

|<-------- 1分間 -------->|
0:30                   1:30 (現在)
     ↑ この範囲のリクエストをカウント
```

**実装**:

```python
import time
from collections import deque

class SlidingWindowLogRateLimiter:
    def __init__(self, limit, window_size):
        self.limit = limit          # 100
        self.window_size = window_size  # 60秒
        self.logs = {}  # {user_id: deque([timestamp, ...])}

    def is_allowed(self, user_id):
        now = time.time()
        window_start = now - self.window_size

        if user_id not in self.logs:
            self.logs[user_id] = deque()

        log = self.logs[user_id]

        # 古いログを削除
        while log and log[0] < window_start:
            log.popleft()

        # 制限内なら許可
        if len(log) < self.limit:
            log.append(now)
            return True

        return False
```

**メリット**:
- 正確（バースト問題なし）

**デメリット**:
- メモリ使用量が多い（全リクエストのタイムスタンプを保持）

### 3. スライディングウィンドウカウンター（Sliding Window Counter）

固定ウィンドウとスライディングログの中間。

```
現在時刻: 1:30 (ウィンドウの50%地点)

前のウィンドウ (0:00-0:59): 80 req
現在のウィンドウ (1:00-1:59): 30 req

推定カウント = 80 × 0.5 + 30 × 1.0 = 40 + 30 = 70 req
```

**実装**:

```python
import time

class SlidingWindowCounterRateLimiter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.counters = {}  # {user_id: {prev_window: count, curr_window: count}}

    def is_allowed(self, user_id):
        now = time.time()
        curr_window = int(now // self.window_size)
        prev_window = curr_window - 1

        # ウィンドウ内の位置（0.0 ~ 1.0）
        window_progress = (now % self.window_size) / self.window_size

        if user_id not in self.counters:
            self.counters[user_id] = {}

        counter = self.counters[user_id]
        prev_count = counter.get(prev_window, 0)
        curr_count = counter.get(curr_window, 0)

        # 加重平均でカウント推定
        estimated_count = prev_count * (1 - window_progress) + curr_count

        if estimated_count < self.limit:
            counter[curr_window] = curr_count + 1
            # 古いウィンドウを削除
            counter.pop(prev_window - 1, None)
            return True

        return False
```

**メリット**:
- メモリ効率が良い（2ウィンドウ分のカウンターのみ）
- バースト問題を軽減

**デメリット**:
- 近似値（100%正確ではない）

### 4. トークンバケット（Token Bucket）

```
バケット容量: 10 トークン
補充レート: 1 トークン/秒

|  バケット  |
|  ○○○○○○○  |  ← 7トークン残っている
|  ○○○     |
|___________|

リクエスト時:
- トークンがあれば1つ消費して許可
- トークンがなければ拒否

時間経過:
- 1秒ごとに1トークン補充（最大10まで）
```

**実装**:

```python
import time

class TokenBucketRateLimiter:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity      # バケット容量: 10
        self.refill_rate = refill_rate  # 補充レート: 1 token/秒
        self.buckets = {}  # {user_id: (tokens, last_update)}

    def is_allowed(self, user_id):
        now = time.time()

        if user_id not in self.buckets:
            self.buckets[user_id] = (self.capacity, now)

        tokens, last_update = self.buckets[user_id]

        # 経過時間に応じてトークン補充
        elapsed = now - last_update
        tokens = min(self.capacity, tokens + elapsed * self.refill_rate)

        if tokens >= 1:
            self.buckets[user_id] = (tokens - 1, now)
            return True

        self.buckets[user_id] = (tokens, now)
        return False
```

**メリット**:
- バーストを許容（溜まったトークン分だけ）
- 平均レートを制御しつつ、一時的なスパイクを許容

**デメリット**:
- パラメータ調整が必要（容量とレート）

**ユースケース**:
- 一時的なバーストを許容したい場合
- AWS API Gateway のデフォルト

### 5. リーキーバケット（Leaky Bucket）

```
バケット容量: 10 リクエスト
流出レート: 1 リクエスト/秒

|  バケット  |
|  ●●●●●●●  |  ← 7リクエストが待機中
|  ●●●     |
|____↓_____|
     ↓ 1 req/秒 で流出（処理）

リクエスト時:
- バケットに空きがあれば追加
- 空きがなければ拒否

処理:
- 一定レートでバケットから取り出して処理
```

**実装**:

```python
import time
from collections import deque

class LeakyBucketRateLimiter:
    def __init__(self, capacity, leak_rate):
        self.capacity = capacity      # バケット容量: 10
        self.leak_rate = leak_rate    # 流出レート: 1 req/秒
        self.buckets = {}  # {user_id: (queue, last_leak)}

    def is_allowed(self, user_id):
        now = time.time()

        if user_id not in self.buckets:
            self.buckets[user_id] = (deque(), now)

        queue, last_leak = self.buckets[user_id]

        # 経過時間に応じてバケットから流出
        elapsed = now - last_leak
        leak_count = int(elapsed * self.leak_rate)
        for _ in range(min(leak_count, len(queue))):
            queue.popleft()

        # バケットに空きがあれば追加
        if len(queue) < self.capacity:
            queue.append(now)
            self.buckets[user_id] = (queue, now)
            return True

        self.buckets[user_id] = (queue, now)
        return False
```

**メリット**:
- 出力レートが一定（スムーズ）
- バースト時のシステム負荷を平準化

**デメリット**:
- バーストが即座に処理されない（キューで待機）

**ユースケース**:
- 一定レートで処理したい場合（バッチ処理への投入など）

### アルゴリズム比較

| アルゴリズム | バースト許容 | メモリ効率 | 実装難易度 | 正確性 |
|-------------|------------|----------|----------|--------|
| 固定ウィンドウ | ❌ 境界で2倍 | ✅ 良い | ✅ 簡単 | ⚪ 普通 |
| スライディングログ | ✅ なし | ❌ 悪い | ⚪ 普通 | ✅ 高い |
| スライディングカウンター | ⚪ 軽減 | ✅ 良い | ⚪ 普通 | ⚪ 近似 |
| トークンバケット | ✅ 制御可能 | ✅ 良い | ⚪ 普通 | ⚪ 普通 |
| リーキーバケット | ❌ キュー待機 | ⚪ 普通 | ⚪ 普通 | ✅ 高い |

---

## 【実装】Redisでの実装

### 固定ウィンドウ（INCR + EXPIRE）

```python
import redis
import time

class RedisFixedWindowRateLimiter:
    def __init__(self, redis_client, limit, window_size):
        self.redis = redis_client
        self.limit = limit
        self.window_size = window_size

    def is_allowed(self, user_id):
        window = int(time.time() // self.window_size)
        key = f"ratelimit:{user_id}:{window}"

        # INCR + EXPIRE をアトミックに
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, self.window_size)
        results = pipe.execute()

        count = results[0]
        return count <= self.limit

# 使用例
redis_client = redis.Redis()
limiter = RedisFixedWindowRateLimiter(redis_client, limit=100, window_size=60)

if limiter.is_allowed('user123'):
    # 処理実行
    pass
else:
    # 429 Too Many Requests
    pass
```

### スライディングウィンドウログ（Sorted Set）

```python
import redis
import time
import uuid

class RedisSlidingWindowLogRateLimiter:
    def __init__(self, redis_client, limit, window_size):
        self.redis = redis_client
        self.limit = limit
        self.window_size = window_size

    def is_allowed(self, user_id):
        now = time.time()
        window_start = now - self.window_size
        key = f"ratelimit:{user_id}"

        pipe = self.redis.pipeline()
        # 古いエントリを削除
        pipe.zremrangebyscore(key, 0, window_start)
        # 現在のカウント取得
        pipe.zcard(key)
        # 新しいエントリ追加
        pipe.zadd(key, {f"{now}:{uuid.uuid4()}": now})
        # TTL設定
        pipe.expire(key, self.window_size)
        results = pipe.execute()

        count = results[1]
        return count < self.limit  # 追加前のカウントで判断
```

### トークンバケット（Lua Script）

```python
import redis
import time

TOKEN_BUCKET_SCRIPT = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_update')
local tokens = tonumber(data[1]) or capacity
local last_update = tonumber(data[2]) or now

-- トークン補充
local elapsed = now - last_update
tokens = math.min(capacity, tokens + elapsed * refill_rate)

-- トークン消費
if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_update', now)
    redis.call('EXPIRE', key, 3600)
    return 1
else
    redis.call('HMSET', key, 'tokens', tokens, 'last_update', now)
    redis.call('EXPIRE', key, 3600)
    return 0
end
"""

class RedisTokenBucketRateLimiter:
    def __init__(self, redis_client, capacity, refill_rate):
        self.redis = redis_client
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.script = self.redis.register_script(TOKEN_BUCKET_SCRIPT)

    def is_allowed(self, user_id, tokens=1):
        key = f"ratelimit:{user_id}"
        now = time.time()
        result = self.script(
            keys=[key],
            args=[self.capacity, self.refill_rate, now, tokens]
        )
        return result == 1
```

---

## 【実装】Nginxでの実装

### limit_req_zone（リーキーバケット）

```nginx
http {
    # Zone定義: 10MBの共有メモリ、キーはIPアドレス
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        location /api/ {
            # バースト許容: 20リクエストまでキューイング
            # nodelay: キューイングせず即座にレスポンス
            limit_req zone=api_limit burst=20 nodelay;

            # 429を返すステータスコード
            limit_req_status 429;

            proxy_pass http://backend;
        }
    }
}
```

### limit_conn（同時接続数制限）

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location /api/ {
            # IPあたり最大10接続
            limit_conn conn_limit 10;
            limit_conn_status 429;

            proxy_pass http://backend;
        }
    }
}
```

### 複合的な制限

```nginx
http {
    # IP単位の制限
    limit_req_zone $binary_remote_addr zone=ip_limit:10m rate=10r/s;

    # APIキー単位の制限
    map $http_x_api_key $api_key_limit_key {
        default $http_x_api_key;
        ""      $binary_remote_addr;
    }
    limit_req_zone $api_key_limit_key zone=apikey_limit:10m rate=100r/s;

    server {
        location /api/ {
            # 両方の制限を適用
            limit_req zone=ip_limit burst=20 nodelay;
            limit_req zone=apikey_limit burst=200 nodelay;

            proxy_pass http://backend;
        }
    }
}
```

---

## 【実装】アプリケーション層での実装

### Python（Flask + Redis）

```python
from flask import Flask, request, jsonify
from functools import wraps
import redis

app = Flask(__name__)
redis_client = redis.Redis()

def rate_limit(limit, window):
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            # キー: IPアドレス + エンドポイント
            key = f"ratelimit:{request.remote_addr}:{request.endpoint}"

            # Redisでカウント
            pipe = redis_client.pipeline()
            pipe.incr(key)
            pipe.expire(key, window)
            results = pipe.execute()

            count = results[0]
            remaining = max(0, limit - count)

            # レスポンスヘッダー
            headers = {
                'X-RateLimit-Limit': str(limit),
                'X-RateLimit-Remaining': str(remaining),
                'X-RateLimit-Reset': str(int(time.time()) + window)
            }

            if count > limit:
                response = jsonify({'error': 'Too Many Requests'})
                response.status_code = 429
                response.headers.extend(headers)
                response.headers['Retry-After'] = str(window)
                return response

            response = f(*args, **kwargs)
            if isinstance(response, tuple):
                response, status = response
                response = jsonify(response)
                response.status_code = status
            response.headers.extend(headers)
            return response
        return wrapper
    return decorator

@app.route('/api/users')
@rate_limit(limit=100, window=60)
def get_users():
    return jsonify({'users': []})
```

### Express.js

```javascript
const express = require('express');
const Redis = require('ioredis');

const app = express();
const redis = new Redis();

function rateLimit(options) {
  const { limit, window } = options;

  return async (req, res, next) => {
    const key = `ratelimit:${req.ip}:${req.path}`;

    const multi = redis.multi();
    multi.incr(key);
    multi.expire(key, window);
    const results = await multi.exec();

    const count = results[0][1];
    const remaining = Math.max(0, limit - count);

    res.set({
      'X-RateLimit-Limit': limit,
      'X-RateLimit-Remaining': remaining,
      'X-RateLimit-Reset': Math.floor(Date.now() / 1000) + window
    });

    if (count > limit) {
      res.set('Retry-After', window);
      return res.status(429).json({ error: 'Too Many Requests' });
    }

    next();
  };
}

app.get('/api/users', rateLimit({ limit: 100, window: 60 }), (req, res) => {
  res.json({ users: [] });
});
```

---

## 【実務】レスポンスヘッダーの設計

### 標準的なヘッダー

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100           # 制限値
X-RateLimit-Remaining: 95        # 残りリクエスト数
X-RateLimit-Reset: 1702540800    # リセット時刻（Unix timestamp）

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1702540800
Retry-After: 30                  # 何秒後にリトライすべきか
```

### クライアント側の対応

```javascript
async function fetchWithRateLimit(url) {
  const response = await fetch(url);

  if (response.status === 429) {
    const retryAfter = response.headers.get('Retry-After');
    const delay = retryAfter ? parseInt(retryAfter) * 1000 : 60000;

    console.log(`Rate limited. Retrying after ${delay}ms`);
    await sleep(delay);
    return fetchWithRateLimit(url);
  }

  // 残りリクエスト数を確認
  const remaining = response.headers.get('X-RateLimit-Remaining');
  if (remaining && parseInt(remaining) < 10) {
    console.warn(`Rate limit warning: ${remaining} requests remaining`);
  }

  return response;
}
```

---

## 【実務】分散環境での課題

### 問題: 複数サーバーでのカウント

```
        ┌─────────────────────────────────────┐
        │          Load Balancer              │
        └──────┬──────────────┬───────────────┘
               │              │
        ┌──────▼────┐  ┌──────▼────┐
        │ Server A  │  │ Server B  │
        │ count: 50 │  │ count: 50 │
        └───────────┘  └───────────┘

User の実際のリクエスト数: 100
各サーバーが認識しているカウント: 50

→ 制限が効いていない！
```

### 解決策: 中央集権的なストア

```
        ┌─────────────────────────────────────┐
        │          Load Balancer              │
        └──────┬──────────────┬───────────────┘
               │              │
        ┌──────▼────┐  ┌──────▼────┐
        │ Server A  │  │ Server B  │
        └─────┬─────┘  └─────┬─────┘
              │              │
              └──────┬───────┘
                     │
              ┌──────▼──────┐
              │    Redis    │  ← 中央でカウント管理
              │  count: 100 │
              └─────────────┘
```

### Redis Cluster での注意点

```python
# キーが同じスロットに配置されるようにする
# {} 内の文字列でハッシュスロットが決まる

key = f"ratelimit:{{user123}}:{window}"  # user123 でスロット決定
```

### レース条件への対策

```python
# 問題のあるコード
count = redis.get(key)
if count < limit:
    redis.incr(key)  # ← この間に他のリクエストが来る可能性

# 解決策1: INCR の結果で判断
count = redis.incr(key)
if count == 1:
    redis.expire(key, window)
if count > limit:
    return False

# 解決策2: Lua Script（アトミック実行）
SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end
return current <= limit and 1 or 0
"""
```

---

## 【実務】多層Rate Limiting

### レイヤーごとの制限

```
                    ┌───────────────────────────┐
                    │       CDN/WAF             │ ← L1: IP単位、粗い制限
                    │  (Cloudflare, AWS WAF)   │    例: 1000 req/min
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      Load Balancer        │ ← L2: Nginx
                    │         (Nginx)           │    例: 100 req/s
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      Application          │ ← L3: ユーザー/API単位
                    │    (Flask + Redis)        │    例: プランに応じた制限
                    └───────────────────────────┘
```

### プラン別の制限

```python
RATE_LIMITS = {
    'free': {'limit': 100, 'window': 3600},       # 100 req/hour
    'basic': {'limit': 1000, 'window': 3600},     # 1000 req/hour
    'pro': {'limit': 10000, 'window': 3600},      # 10000 req/hour
    'enterprise': {'limit': 100000, 'window': 3600},  # 100000 req/hour
}

def get_rate_limit(user):
    plan = user.subscription_plan
    return RATE_LIMITS.get(plan, RATE_LIMITS['free'])
```

### エンドポイント別の制限

```python
ENDPOINT_LIMITS = {
    'POST /api/auth/login': {'limit': 5, 'window': 60},       # ブルートフォース対策
    'POST /api/auth/register': {'limit': 3, 'window': 3600},  # スパム対策
    'GET /api/search': {'limit': 30, 'window': 60},           # 重いクエリ
    'default': {'limit': 100, 'window': 60},
}

def get_endpoint_limit(method, path):
    key = f'{method} {path}'
    return ENDPOINT_LIMITS.get(key, ENDPOINT_LIMITS['default'])
```

---

## 【実務】監視とアラート

### 監視すべきメトリクス

| メトリクス | 説明 | アラート閾値（例） |
|-----------|------|------------------|
| 429 レスポンス率 | Rate Limited されたリクエストの割合 | > 5% |
| 制限到達ユーザー数 | 制限に達したユーザーの数 | 急増時 |
| 平均リクエストレート | ユーザーあたりの平均レート | 異常値検出 |

### Prometheus メトリクス

```python
from prometheus_client import Counter, Histogram

rate_limit_hits = Counter(
    'rate_limit_hits_total',
    'Total number of rate limited requests',
    ['endpoint', 'user_tier']
)

rate_limit_remaining = Histogram(
    'rate_limit_remaining',
    'Remaining rate limit when request was made',
    ['endpoint']
)

@app.after_request
def record_metrics(response):
    if response.status_code == 429:
        rate_limit_hits.labels(
            endpoint=request.endpoint,
            user_tier=g.user.tier if hasattr(g, 'user') else 'anonymous'
        ).inc()

    remaining = response.headers.get('X-RateLimit-Remaining')
    if remaining:
        rate_limit_remaining.labels(endpoint=request.endpoint).observe(int(remaining))

    return response
```

### アラート例（Alertmanager）

```yaml
groups:
  - name: rate_limiting
    rules:
      - alert: HighRateLimitRate
        expr: |
          sum(rate(rate_limit_hits_total[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Rate limiting is affecting more than 5% of requests"

      - alert: RateLimitAbuse
        expr: |
          rate(rate_limit_hits_total{user_tier="free"}[1h]) > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Potential rate limit abuse detected"
```

---

## 実務チェックリスト

### 設計時

- [ ] どのアルゴリズムを使うか決めたか
- [ ] 制限の粒度は適切か（IP/ユーザー/APIキー）
- [ ] エンドポイントごとの制限は必要か
- [ ] プラン別の制限は必要か

### 実装時

- [ ] 分散環境での整合性は確保されているか
- [ ] レスポンスヘッダーは適切か
- [ ] 429 レスポンスの内容は親切か
- [ ] Retry-After ヘッダーは返しているか

### 運用時

- [ ] 429 レスポンス率は監視されているか
- [ ] アラートは設定されているか
- [ ] 制限値の調整は容易か
- [ ] 特定ユーザーの制限緩和は可能か

---

## まとめ

Rate Limitingの本質は、**リソースの公平な分配とシステムの保護**だ。

### アルゴリズムの選択

| 要件 | アルゴリズム |
|------|-------------|
| シンプルさ重視 | 固定ウィンドウ |
| 正確性重視 | スライディングログ |
| バランス型 | スライディングカウンター |
| バースト許容 | トークンバケット |
| 安定出力 | リーキーバケット |

### 実装の選択

| 要件 | 実装 |
|------|------|
| シンプル、単一サーバー | メモリ内 |
| 分散環境 | Redis |
| インフラ層 | Nginx / API Gateway |
| アプリ層 | ミドルウェア |

### 運用のポイント

- 適切なエラーレスポンス（429 + Retry-After）
- 多層防御（CDN → LB → App）
- 監視とアラート

**Rate Limitingは保険。平時は目立たないが、いざという時にシステムを救う。**
