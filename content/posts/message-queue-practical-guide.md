---
title: "メッセージキュー実践ガイド：Q4MからKafkaまで、非同期処理の設計と運用"
date: 2025-12-13
draft: false
tags: ["メッセージキュー", "Redis", "Kafka", "RabbitMQ", "Q4M", "非同期処理", "システム設計", "実務"]
description: "同期処理の限界を超えるメッセージキューの設計・実装・運用。Q4M、Redis、RabbitMQ、Kafka、SQSの比較と使い分け、障害対応まで完全網羅"
cover:
  image: "images/covers/message-queue.jpg"
  alt: "メッセージキュー実践ガイド"
  relative: false
---

## この記事の対象読者

- 「メッセージキュー」を聞いたことはあるけど、使ったことがない人
- 同期処理の限界を感じている人
- レガシーなキュー（Q4M等）を運用していて、モダン化を考えている人
- Kafka、RabbitMQ、SQSの違いがよく分からない人

この記事では、**メッセージキューの基本概念**から、**各ミドルウェアの比較**、**実装パターン**、**障害対応**、**レガシーからの移行**まで、体系的に解説します。

---

## なぜメッセージキューが必要なのか

### 同期処理の限界

```
【同期処理】
ユーザー → API → 重い処理（メール送信、画像変換、外部API）→ レスポンス
                      ↓
              ユーザーは待ち続ける（5秒、10秒...）
```

**問題点：**
- ユーザー体験が悪い（待たされる）
- タイムアウトのリスク
- 1つの処理が詰まると全体に影響
- スケールしにくい

### 非同期処理で解決

```
【非同期処理（メッセージキュー）】
ユーザー → API → キューに投入 → 即座にレスポンス（"受け付けました"）
                      ↓
              ワーカーが後から処理（メール送信、画像変換等）
```

**メリット：**
- ユーザーを待たせない
- 処理を分散できる
- 障害に強い（リトライできる）
- スケールしやすい

---

## メッセージキューの基本概念

### アーキテクチャ

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│  Producer   │────→│  Message Queue  │────→│  Consumer   │
│ （送信側）    │     │   （キュー）      │     │  （受信側）   │
└─────────────┘     └─────────────────┘     └─────────────┘
      ↓                     ↓                     ↓
  メッセージを          メッセージを           メッセージを
  キューに投入          一時的に保存           取り出して処理
```

### 用語解説

| 用語 | 説明 |
|------|------|
| **Producer** | メッセージを送信する側 |
| **Consumer** | メッセージを受信・処理する側 |
| **Message** | キューに入るデータ（JSON等） |
| **Queue** | メッセージを保持するバッファ |
| **Broker** | キューを管理するサーバー |
| **Ack** | 処理完了の確認応答 |

### 2つのメッセージングパターン

#### 1. Point-to-Point（1対1）

```
Producer → Queue → Consumer

- 1つのメッセージは1つのConsumerだけが処理
- 処理が完了したらメッセージは削除
- 用途：タスク処理、ジョブキュー
```

#### 2. Pub/Sub（1対多）

```
Publisher → Topic → Subscriber A
                  → Subscriber B
                  → Subscriber C

- 1つのメッセージを複数のSubscriberが受信
- 用途：イベント通知、ログ配信、リアルタイム更新
```

---

## メッセージキューの比較

### 主要なメッセージキュー一覧

| 名前 | 特徴 | 用途 |
|------|------|------|
| **Redis (List/Stream)** | シンプル、高速 | 軽量なジョブキュー |
| **RabbitMQ** | 柔軟なルーティング | 複雑なメッセージング |
| **Apache Kafka** | 高スループット、永続化 | ログ収集、イベント駆動 |
| **Amazon SQS** | フルマネージド | AWSインフラ |
| **Q4M** | MySQL上で動作 | レガシーシステム |

### 詳細比較表

| 項目 | Redis | RabbitMQ | Kafka | SQS | Q4M |
|------|-------|----------|-------|-----|-----|
| **スループット** | 高 | 中 | 非常に高 | 中 | 低〜中 |
| **永続化** | オプション | あり | あり | あり | あり(MySQL) |
| **順序保証** | あり | あり | パーティション内 | なし(FIFO除く) | あり |
| **Pub/Sub** | あり | あり | あり | SNS連携 | なし |
| **運用難易度** | 低 | 中 | 高 | 低(マネージド) | 低 |
| **遅延配信** | 手動実装 | あり | なし | あり | 手動実装 |
| **現在の推奨** | ✅ | ✅ | ✅ | ✅ | ❌(レガシー) |

---

# 第1部：Q4M（レガシーだが現役）

## Q4Mとは

**Q4M（Queue for MySQL）** は、MySQLのストレージエンジンとして実装されたメッセージキューです。

```
特徴：
- MySQLのテーブルがそのままキューになる
- SQLでキュー操作ができる
- 2008年頃に開発、現在は開発停止
- レガシーシステムではまだ現役
```

## なぜQ4Mが使われたのか

```
2008年頃の状況：
- RabbitMQはまだ発展途上
- Kafkaは存在しない（2011年〜）
- Redisもキューとしては使われていなかった
- MySQLは既に多くの会社で運用されていた

→ 「MySQLに追加するだけでキューが使える」Q4Mは魅力的だった
```

## Q4Mの仕組み

### テーブル作成

```sql
-- Q4Mエンジンでテーブルを作成
CREATE TABLE job_queue (
    id INT AUTO_INCREMENT PRIMARY KEY,
    job_type VARCHAR(50) NOT NULL,
    payload TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=QUEUE;
```

### メッセージの投入（Producer）

```sql
-- 通常のINSERTでキューに投入
INSERT INTO job_queue (job_type, payload)
VALUES ('send_email', '{"to":"user@example.com","subject":"Welcome"}');
```

### メッセージの取得（Consumer）

```sql
-- queue_wait()でメッセージを待機・取得
SELECT * FROM job_queue WHERE queue_wait('job_queue', 10);
-- 10秒間待機、メッセージがあれば返す

-- 処理が完了したらqueue_end()で確定
SELECT queue_end();

-- 処理に失敗したらqueue_abort()でキューに戻す
SELECT queue_abort();
```

## Q4Mのワーカー実装（PHP）

```php
<?php
// worker.php - Q4Mワーカーの実装例

class Q4MWorker
{
    private PDO $pdo;
    private bool $running = true;

    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
        // シグナルハンドラ設定
        pcntl_signal(SIGTERM, [$this, 'handleSignal']);
        pcntl_signal(SIGINT, [$this, 'handleSignal']);
    }

    public function handleSignal(int $signal): void
    {
        echo "Received signal {$signal}, shutting down...\n";
        $this->running = false;
    }

    public function run(): void
    {
        echo "Worker started\n";

        while ($this->running) {
            pcntl_signal_dispatch();

            try {
                $this->processJob();
            } catch (Exception $e) {
                echo "Error: " . $e->getMessage() . "\n";
                sleep(1);
            }
        }

        echo "Worker stopped\n";
    }

    private function processJob(): void
    {
        // queue_wait()でジョブを取得（10秒タイムアウト）
        $stmt = $this->pdo->query(
            "SELECT * FROM job_queue WHERE queue_wait('job_queue', 10)"
        );
        $job = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$job) {
            // タイムアウト（ジョブなし）
            return;
        }

        echo "Processing job: {$job['id']} - {$job['job_type']}\n";

        try {
            $this->handleJob($job);

            // 処理成功 → queue_end()で削除
            $this->pdo->query("SELECT queue_end()");
            echo "Job {$job['id']} completed\n";

        } catch (Exception $e) {
            // 処理失敗 → queue_abort()でキューに戻す
            $this->pdo->query("SELECT queue_abort()");
            echo "Job {$job['id']} failed: " . $e->getMessage() . "\n";
        }
    }

    private function handleJob(array $job): void
    {
        $payload = json_decode($job['payload'], true);

        switch ($job['job_type']) {
            case 'send_email':
                $this->sendEmail($payload);
                break;
            case 'process_image':
                $this->processImage($payload);
                break;
            default:
                throw new Exception("Unknown job type: {$job['job_type']}");
        }
    }

    private function sendEmail(array $payload): void
    {
        // メール送信処理
        mail($payload['to'], $payload['subject'], $payload['body'] ?? '');
    }

    private function processImage(array $payload): void
    {
        // 画像処理
        // ...
    }
}

// 実行
$pdo = new PDO('mysql:host=localhost;dbname=myapp', 'user', 'pass');
$worker = new Q4MWorker($pdo);
$worker->run();
```

## Q4Mの監視

```sql
-- キューの状態を確認
SELECT queue_stats('job_queue');

-- 滞留しているジョブ数
SELECT COUNT(*) FROM job_queue;

-- 古いジョブの確認
SELECT * FROM job_queue
WHERE created_at < NOW() - INTERVAL 1 HOUR
ORDER BY created_at
LIMIT 10;
```

## Q4Mの問題点

| 問題 | 説明 |
|------|------|
| **開発停止** | 2012年頃から更新なし |
| **MySQL依存** | MySQLのバージョンアップで動かなくなる可能性 |
| **スケール限界** | MySQLの性能がボトルネック |
| **機能不足** | 遅延配信、優先度、Pub/Subなし |
| **運用ノウハウ** | ドキュメントが少ない |

## Q4Mからの移行

```
Q4Mを使い続けるリスク：
- MySQLバージョンアップ時の互換性問題
- 障害時の対応が難しい（ノウハウがない）
- エンジニア採用時に説明が必要

移行先の候補：
1. Redis（シンプルな置き換え）
2. Amazon SQS（マネージドで運用楽）
3. RabbitMQ（機能豊富）
```

---

# 第2部：Redis でのキュー実装

## Redisがキューに適している理由

```
✅ インメモリで高速
✅ 導入が簡単（多くの環境で既に使っている）
✅ List / Stream で柔軟な実装が可能
✅ Pub/Subもサポート
```

## 方式1：List を使ったシンプルなキュー

### 基本操作

```bash
# キューに投入（右から追加）
RPUSH job_queue '{"type":"send_email","to":"user@example.com"}'

# キューから取得（左から取得、ブロッキング）
BLPOP job_queue 10
# 10秒待機、メッセージがあれば返す

# キューの長さ
LLEN job_queue
```

### Pythonでの実装

```python
import redis
import json
import signal
import sys

class RedisQueueWorker:
    def __init__(self, redis_client, queue_name):
        self.redis = redis_client
        self.queue_name = queue_name
        self.running = True

        signal.signal(signal.SIGTERM, self.handle_signal)
        signal.signal(signal.SIGINT, self.handle_signal)

    def handle_signal(self, signum, frame):
        print(f"Received signal {signum}, shutting down...")
        self.running = False

    def enqueue(self, job_type: str, payload: dict):
        """ジョブをキューに投入"""
        message = json.dumps({
            "type": job_type,
            "payload": payload
        })
        self.redis.rpush(self.queue_name, message)

    def run(self):
        """ワーカーループ"""
        print(f"Worker started, listening on {self.queue_name}")

        while self.running:
            try:
                # BLPOP: ブロッキングで取得（10秒タイムアウト）
                result = self.redis.blpop(self.queue_name, timeout=10)

                if result is None:
                    continue  # タイムアウト

                _, message = result
                job = json.loads(message)

                print(f"Processing: {job['type']}")
                self.process_job(job)
                print(f"Completed: {job['type']}")

            except Exception as e:
                print(f"Error: {e}")
                # エラー時はリトライキューに入れる等の処理

        print("Worker stopped")

    def process_job(self, job: dict):
        job_type = job["type"]
        payload = job["payload"]

        if job_type == "send_email":
            self.send_email(payload)
        elif job_type == "process_image":
            self.process_image(payload)
        else:
            raise ValueError(f"Unknown job type: {job_type}")

    def send_email(self, payload):
        print(f"Sending email to {payload['to']}")
        # 実際のメール送信処理

    def process_image(self, payload):
        print(f"Processing image: {payload['path']}")
        # 実際の画像処理

# 使用例
if __name__ == "__main__":
    r = redis.Redis(host='localhost', port=6379, db=0)
    worker = RedisQueueWorker(r, "job_queue")

    # ジョブを投入
    worker.enqueue("send_email", {"to": "user@example.com", "subject": "Hello"})

    # ワーカー起動
    worker.run()
```

## 方式2：Reliable Queue パターン

シンプルなBLPOPには問題があります：

```
問題：処理中にワーカーが死ぬとメッセージが消える

BLPOP → メッセージ取得 → ワーカーがクラッシュ → メッセージ消失
```

**解決策：BRPOPLPUSH（または BLMOVE）**

```python
class ReliableQueue:
    def __init__(self, redis_client, queue_name):
        self.redis = redis_client
        self.queue_name = queue_name
        self.processing_queue = f"{queue_name}:processing"

    def enqueue(self, message: str):
        self.redis.rpush(self.queue_name, message)

    def dequeue(self, timeout=10):
        """取得と同時に処理中キューに移動"""
        # BRPOPLPUSH: 取得しつつprocessing_queueに移動
        message = self.redis.brpoplpush(
            self.queue_name,
            self.processing_queue,
            timeout=timeout
        )
        return message

    def complete(self, message: str):
        """処理完了 → processing_queueから削除"""
        self.redis.lrem(self.processing_queue, 1, message)

    def fail(self, message: str):
        """処理失敗 → 元のキューに戻す"""
        self.redis.lrem(self.processing_queue, 1, message)
        self.redis.rpush(self.queue_name, message)

    def recover_stale_jobs(self, max_age_seconds=300):
        """処理中のまま放置されたジョブを回復"""
        # 実装では、各メッセージにタイムスタンプを含め、
        # 古いものをqueue_nameに戻す
        pass
```

## 方式3：Redis Streams（推奨）

Redis 5.0以降で使える**Streams**は、Kafkaライクな機能を持ちます。

```bash
# メッセージを追加
XADD job_stream * type send_email to user@example.com

# Consumer Groupを作成
XGROUP CREATE job_stream workers $ MKSTREAM

# メッセージを読み取り（Consumer Group経由）
XREADGROUP GROUP workers worker1 COUNT 1 BLOCK 10000 STREAMS job_stream >

# 処理完了を通知（ACK）
XACK job_stream workers 1234567890123-0
```

### Pythonでの実装

```python
import redis
import json

class RedisStreamWorker:
    def __init__(self, redis_client, stream_name, group_name, consumer_name):
        self.redis = redis_client
        self.stream = stream_name
        self.group = group_name
        self.consumer = consumer_name
        self.running = True

        # Consumer Groupを作成（既に存在すればスキップ）
        try:
            self.redis.xgroup_create(stream_name, group_name, id='0', mkstream=True)
        except redis.ResponseError as e:
            if "BUSYGROUP" not in str(e):
                raise

    def enqueue(self, job_type: str, payload: dict):
        """Streamにメッセージを追加"""
        self.redis.xadd(self.stream, {
            "type": job_type,
            "payload": json.dumps(payload)
        })

    def run(self):
        print(f"Worker {self.consumer} started")

        while self.running:
            try:
                # XREADGROUPでメッセージを取得
                messages = self.redis.xreadgroup(
                    self.group,
                    self.consumer,
                    {self.stream: '>'},  # 新しいメッセージのみ
                    count=1,
                    block=10000  # 10秒ブロック
                )

                if not messages:
                    continue

                for stream_name, stream_messages in messages:
                    for message_id, data in stream_messages:
                        self.process_message(message_id, data)

            except Exception as e:
                print(f"Error: {e}")

    def process_message(self, message_id, data):
        try:
            job_type = data[b'type'].decode()
            payload = json.loads(data[b'payload'].decode())

            print(f"Processing {message_id}: {job_type}")

            # ジョブを処理
            self.handle_job(job_type, payload)

            # 処理完了 → ACK
            self.redis.xack(self.stream, self.group, message_id)
            print(f"Completed {message_id}")

        except Exception as e:
            print(f"Failed {message_id}: {e}")
            # 失敗したメッセージはACKしない → 再処理される

    def handle_job(self, job_type, payload):
        if job_type == "send_email":
            print(f"Sending email to {payload['to']}")
        # ...

    def recover_pending(self):
        """処理中のまま残っているメッセージを再処理"""
        pending = self.redis.xpending_range(
            self.stream, self.group, '-', '+', count=100
        )
        for item in pending:
            message_id = item['message_id']
            idle_time = item['time_since_delivered']

            if idle_time > 300000:  # 5分以上経過
                # 自分のConsumerに割り当て直す
                self.redis.xclaim(
                    self.stream, self.group, self.consumer,
                    min_idle_time=300000,
                    message_ids=[message_id]
                )

# 使用例
r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=False)
worker = RedisStreamWorker(r, "job_stream", "workers", "worker1")

# ジョブ投入
worker.enqueue("send_email", {"to": "user@example.com"})

# ワーカー起動
worker.run()
```

---

# 第3部：RabbitMQ

## RabbitMQの特徴

```
✅ 柔軟なルーティング（Exchange）
✅ 複数のメッセージングパターン
✅ 永続化、クラスタリング対応
✅ 管理UI付き
```

## RabbitMQのアーキテクチャ

```
Producer → Exchange → Binding → Queue → Consumer
              ↓
      ルーティングルール
```

### Exchange の種類

| 種類 | ルーティング |
|------|-------------|
| **Direct** | ルーティングキーが完全一致 |
| **Fanout** | 全てのキューにブロードキャスト |
| **Topic** | パターンマッチング（*.error, log.#） |
| **Headers** | ヘッダー属性でマッチング |

## Pythonでの実装

```python
import pika
import json

class RabbitMQProducer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host=host)
        )
        self.channel = self.connection.channel()

        # キューを宣言（存在しなければ作成）
        self.channel.queue_declare(queue='job_queue', durable=True)

    def publish(self, job_type: str, payload: dict):
        message = json.dumps({
            "type": job_type,
            "payload": payload
        })

        self.channel.basic_publish(
            exchange='',
            routing_key='job_queue',
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2,  # 永続化
            )
        )
        print(f"Sent: {job_type}")

    def close(self):
        self.connection.close()


class RabbitMQConsumer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host=host)
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue='job_queue', durable=True)

        # 1つずつ処理（prefetch=1）
        self.channel.basic_qos(prefetch_count=1)

    def callback(self, ch, method, properties, body):
        job = json.loads(body)
        print(f"Processing: {job['type']}")

        try:
            self.process_job(job)
            # 処理成功 → ACK
            ch.basic_ack(delivery_tag=method.delivery_tag)
            print(f"Completed: {job['type']}")

        except Exception as e:
            print(f"Failed: {e}")
            # 処理失敗 → NACK（リキュー）
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

    def process_job(self, job):
        job_type = job["type"]
        payload = job["payload"]

        if job_type == "send_email":
            print(f"Sending email to {payload['to']}")
        # ...

    def run(self):
        self.channel.basic_consume(
            queue='job_queue',
            on_message_callback=self.callback
        )
        print("Waiting for messages...")
        self.channel.start_consuming()

# Producer
producer = RabbitMQProducer()
producer.publish("send_email", {"to": "user@example.com"})
producer.close()

# Consumer
consumer = RabbitMQConsumer()
consumer.run()
```

## 遅延キュー（Dead Letter Exchange）

```python
# 遅延キューの設定
channel.exchange_declare(exchange='delayed_exchange', exchange_type='direct')

# 本来のキュー
channel.queue_declare(
    queue='job_queue',
    durable=True,
    arguments={
        'x-dead-letter-exchange': 'delayed_exchange',
        'x-dead-letter-routing-key': 'delayed'
    }
)

# 遅延用キュー（TTL付き）
channel.queue_declare(
    queue='delayed_queue',
    durable=True,
    arguments={
        'x-message-ttl': 60000,  # 60秒後に配信
        'x-dead-letter-exchange': '',
        'x-dead-letter-routing-key': 'job_queue'
    }
)
```

---

# 第4部：Apache Kafka

## Kafkaの特徴

```
✅ 超高スループット（100万メッセージ/秒）
✅ 永続化・リプレイ可能
✅ 分散・スケーラブル
✅ イベントソーシングに最適
```

## Kafkaのアーキテクチャ

```
┌────────────────────────────────────────────────────────┐
│                      Kafka Cluster                     │
│  ┌─────────────────────────────────────────────────┐  │
│  │                    Topic                         │  │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐      │  │
│  │  │Partition 0│ │Partition 1│ │Partition 2│      │  │
│  │  │  [0][1][2]│ │  [0][1]   │ │  [0][1][2]│      │  │
│  │  └───────────┘ └───────────┘ └───────────┘      │  │
│  └─────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
        ↑                                    ↓
    Producer                           Consumer Group
    (書き込み)                          (読み取り)
```

### Kafkaの概念

| 概念 | 説明 |
|------|------|
| **Topic** | メッセージのカテゴリ |
| **Partition** | Topicの分割単位（並列処理用） |
| **Offset** | Partition内のメッセージ位置 |
| **Consumer Group** | Consumerの論理グループ |
| **Broker** | Kafkaサーバー |

## Pythonでの実装

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# メッセージ送信
producer.send('job_topic', {
    "type": "send_email",
    "payload": {"to": "user@example.com"}
})
producer.flush()

# Consumer
consumer = KafkaConsumer(
    'job_topic',
    bootstrap_servers=['localhost:9092'],
    group_id='job_workers',
    auto_offset_reset='earliest',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    job = message.value
    print(f"Processing: {job['type']}")
    # 処理
    # Consumer Groupにより自動的にオフセットがコミットされる
```

## Kafkaの使いどころ

```
✅ ログ収集・分析パイプライン
✅ イベント駆動アーキテクチャ
✅ マイクロサービス間通信
✅ リアルタイムデータストリーミング
✅ 監査ログ（永続化が必要）

❌ 単純なジョブキュー（オーバースペック）
❌ 小規模システム（運用コストが高い）
```

---

# 第5部：Amazon SQS

## SQSの特徴

```
✅ フルマネージド（運用不要）
✅ 高い可用性（マルチAZ）
✅ 自動スケーリング
✅ 従量課金
```

## 2つのキュータイプ

| タイプ | 特徴 | 用途 |
|--------|------|------|
| **Standard** | 高スループット、順序保証なし | 一般的なジョブ |
| **FIFO** | 順序保証あり、重複排除 | 順序が重要な処理 |

## Pythonでの実装（boto3）

```python
import boto3
import json

sqs = boto3.client('sqs', region_name='ap-northeast-1')
queue_url = 'https://sqs.ap-northeast-1.amazonaws.com/123456789/job-queue'

# Producer
def send_message(job_type: str, payload: dict):
    sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps({
            "type": job_type,
            "payload": payload
        }),
        MessageAttributes={
            'JobType': {
                'StringValue': job_type,
                'DataType': 'String'
            }
        }
    )

# Consumer
def process_messages():
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,  # ロングポーリング
            VisibilityTimeout=60  # 処理時間
        )

        messages = response.get('Messages', [])

        for message in messages:
            try:
                job = json.loads(message['Body'])
                print(f"Processing: {job['type']}")

                # ジョブ処理
                handle_job(job)

                # 処理成功 → 削除
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )

            except Exception as e:
                print(f"Error: {e}")
                # VisibilityTimeout後に再処理される

def handle_job(job):
    if job["type"] == "send_email":
        print(f"Sending email to {job['payload']['to']}")

# 使用例
send_message("send_email", {"to": "user@example.com"})
process_messages()
```

## Dead Letter Queue（DLQ）

```python
# 失敗したメッセージを別のキューに移動
# AWSコンソールまたはCloudFormationで設定

# DLQからメッセージを取得して分析
dlq_url = 'https://sqs.../job-queue-dlq'

response = sqs.receive_message(
    QueueUrl=dlq_url,
    MaxNumberOfMessages=10
)

for message in response.get('Messages', []):
    print(f"Failed message: {message['Body']}")
    # 原因を調査し、修正後に再投入
```

---

# 第6部：実装パターン集

## パターン1：リトライ with 指数バックオフ

```python
import time
import random

def process_with_retry(job, max_retries=3):
    for attempt in range(max_retries):
        try:
            process_job(job)
            return True
        except Exception as e:
            if attempt == max_retries - 1:
                # 最後の試行 → DLQに入れる
                send_to_dlq(job, str(e))
                return False

            # 指数バックオフ + ジッター
            wait_time = (2 ** attempt) + random.uniform(0, 1)
            print(f"Retry {attempt + 1} after {wait_time:.2f}s")
            time.sleep(wait_time)
```

## パターン2：優先度付きキュー

```python
# Redisで優先度別のキューを作成
QUEUES = [
    "jobs:critical",  # 最優先
    "jobs:high",
    "jobs:normal",
    "jobs:low"
]

def dequeue_by_priority(redis_client, timeout=10):
    """優先度順にデキュー"""
    result = redis_client.blpop(QUEUES, timeout=timeout)
    if result:
        queue_name, message = result
        priority = queue_name.split(':')[1]
        return priority, message
    return None, None

def enqueue_with_priority(redis_client, priority, message):
    """優先度を指定してエンキュー"""
    queue_name = f"jobs:{priority}"
    redis_client.rpush(queue_name, message)
```

## パターン3：バッチ処理

```python
def process_batch(redis_client, batch_size=100, max_wait=5):
    """複数メッセージをまとめて処理"""
    messages = []
    start_time = time.time()

    while len(messages) < batch_size:
        elapsed = time.time() - start_time
        remaining = max(0, max_wait - elapsed)

        if remaining == 0:
            break

        result = redis_client.blpop("jobs", timeout=int(remaining))
        if result:
            messages.append(result[1])

    if messages:
        # バッチ処理
        process_messages_bulk(messages)
        print(f"Processed {len(messages)} messages")
```

## パターン4：遅延キュー

```python
import time

def enqueue_delayed(redis_client, message, delay_seconds):
    """指定秒後に処理されるジョブを投入"""
    execute_at = time.time() + delay_seconds
    redis_client.zadd("delayed_jobs", {message: execute_at})

def process_delayed_jobs(redis_client):
    """遅延ジョブを処理"""
    while True:
        now = time.time()

        # 実行時刻を過ぎたジョブを取得
        jobs = redis_client.zrangebyscore(
            "delayed_jobs", 0, now, start=0, num=10
        )

        for job in jobs:
            # 遅延キューから削除
            if redis_client.zrem("delayed_jobs", job):
                # 通常キューに投入
                redis_client.rpush("jobs", job)

        time.sleep(1)
```

## パターン5：冪等性の確保

```python
def process_idempotent(redis_client, job):
    """同じジョブを2回処理しない"""
    job_id = job["id"]
    lock_key = f"processed:{job_id}"

    # 既に処理済みかチェック
    if redis_client.exists(lock_key):
        print(f"Job {job_id} already processed, skipping")
        return

    # 処理を実行
    try:
        do_actual_work(job)

        # 処理済みフラグを立てる（24時間で期限切れ）
        redis_client.setex(lock_key, 86400, "1")

    except Exception as e:
        # 失敗時はフラグを立てない → リトライ可能
        raise
```

---

# 第7部：障害対応

## よくある障害パターン

### 1. キューが詰まる

```bash
# 症状：キューの長さが増え続ける

# 確認
redis-cli LLEN job_queue
# → 100000 (通常は100程度)

# 原因
# - Consumerが死んでいる
# - 処理が遅い
# - 大量のジョブが一気に投入された
```

**対応：**

```bash
# 1. Consumerの状態を確認
ps aux | grep worker

# 2. Consumerを増やす
./worker.py &
./worker.py &
./worker.py &

# 3. 緊急時：キューを別の場所に退避
redis-cli RENAME job_queue job_queue_backup
# → 新しいジョブは空のキューへ
# → 後でbackupを処理
```

### 2. メッセージの重複処理

```
症状：同じメールが2通送られた

原因：
- At-least-once配信（ACKが失われた）
- Consumerの再起動
```

**対応：**

```python
# 冪等性を確保する
def send_email_idempotent(job_id, to, subject, body):
    # 送信済みかチェック
    if is_email_sent(job_id):
        return  # スキップ

    # メール送信
    send_email(to, subject, body)

    # 送信済みフラグを保存
    mark_email_sent(job_id)
```

### 3. メッセージの消失

```
症状：投入したはずのジョブが処理されない

原因：
- Redisの永続化設定ミス
- ACK後にConsumerがクラッシュ
- キューの誤削除
```

**対応：**

```bash
# Redis永続化設定の確認
redis-cli CONFIG GET appendonly
# → "yes" であるべき

redis-cli CONFIG GET save
# → 適切なスナップショット間隔

# 永続化を有効にする
redis-cli CONFIG SET appendonly yes
```

### 4. Consumer が処理中にクラッシュ

```
症状：ジョブが消えた or 無限ループ
```

**対応：**

```python
# Reliable Queue パターンを使用
# 処理中キューを監視し、放置されたジョブを回復

def recover_orphaned_jobs(redis_client, max_age=300):
    """処理中のまま放置されたジョブを回復"""
    processing_queue = "jobs:processing"
    main_queue = "jobs"

    # 処理中キューの全メッセージを確認
    # （実際にはタイムスタンプ付きで管理）
    orphaned = redis_client.lrange(processing_queue, 0, -1)

    for job in orphaned:
        job_data = json.loads(job)
        if is_stale(job_data, max_age):
            # 元のキューに戻す
            redis_client.lrem(processing_queue, 1, job)
            redis_client.rpush(main_queue, job)
            print(f"Recovered orphaned job: {job_data['id']}")
```

## 監視すべきメトリクス

```yaml
# Prometheus アラート例
groups:
  - name: message_queue
    rules:
      # キューの長さ
      - alert: QueueBacklogHigh
        expr: redis_list_length{list="job_queue"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Job queue backlog is high"

      # Consumer数
      - alert: NoActiveConsumers
        expr: count(up{job="queue_worker"}) == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No active queue workers"

      # 処理レート
      - alert: ProcessingRateLow
        expr: rate(jobs_processed_total[5m]) < 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Job processing rate is low"
```

---

## 選び方ガイド

### フローチャート

```
                    ┌─────────────────┐
                    │ メッセージキューが必要 │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │ 既存インフラは？              │
              └──────────────┬──────────────┘
                    ┌────────┴────────┐
                    ↓                 ↓
               MySQL系            クラウド
                    │                 │
                    ↓                 ↓
          ┌────────────────┐  ┌────────────────┐
          │ Q4M使用中？     │  │ AWS？          │
          └───────┬────────┘  └───────┬────────┘
                  │                   │
           Yes    │    No      Yes    │    No
            ↓     ↓             ↓     ↓
    ┌───────────┐ │     ┌────────┐    │
    │移行を検討  │ │     │ SQS   │    │
    └───────────┘ │     └────────┘    │
                  ↓                   ↓
          ┌─────────────┐     ┌─────────────┐
          │ 要件は？    │     │ 要件は？    │
          └──────┬──────┘     └──────┬──────┘
                 │                   │
    ┌────────────┼────────────┐     │
    ↓            ↓            ↓     ↓
 シンプル    高機能      超高負荷   同左
    ↓            ↓            ↓
 ┌─────┐   ┌─────────┐   ┌───────┐
 │Redis│   │RabbitMQ │   │ Kafka │
 └─────┘   └─────────┘   └───────┘
```

### 推奨まとめ

| ケース | 推奨 |
|--------|------|
| 小〜中規模、Redisあり | **Redis Streams** |
| AWS環境 | **Amazon SQS** |
| 複雑なルーティング | **RabbitMQ** |
| 超高スループット、イベント駆動 | **Kafka** |
| レガシーMySQL環境 | Q4M（移行を計画） |

---

## まとめ

### メッセージキューの本質

```
1. 同期→非同期でユーザー体験向上
2. 処理を分離してスケーラビリティ向上
3. 障害に強い設計（リトライ、DLQ）
4. ピーク負荷の吸収
```

### 各キューの選び方

| キュー | 一言 |
|--------|------|
| Q4M | レガシー、移行を検討 |
| Redis | シンプル、軽量 |
| RabbitMQ | 柔軟、多機能 |
| Kafka | 大規模、永続化 |
| SQS | マネージド、楽 |

### 実装の心がけ

```
1. 冪等性を確保する
2. リトライ設計を入れる
3. DLQで失敗を補足する
4. 監視とアラートを設定する
5. 処理中メッセージの回復機構
```

---

## 参考リンク

- [Redis Streams公式](https://redis.io/docs/data-types/streams/)
- [RabbitMQ公式チュートリアル](https://www.rabbitmq.com/getstarted.html)
- [Apache Kafka公式](https://kafka.apache.org/documentation/)
- [Amazon SQS開発者ガイド](https://docs.aws.amazon.com/sqs/)
- [Q4M (GitHub)](https://github.com/q4m/q4m)
