---
title: "AWSなしでここまでできる：Raspberry Pi × MQTT × Webhookで現場通知を最速に作る方法"
date: 2025-12-20
draft: false
categories: ["IoT・自動化"]
tags: ["Raspberry Pi", "MQTT", "Webhook", "IoT", "Slack", "自動化", "受託開発"]
description: "Mac/スマホからpub → Piでsub → Slack通知 → systemd常駐。クラウド不要で始める現場通知システムの最小構成を、コピペで動くコードと共に解説します。"
cover:
  image: "images/covers/raspberry-pi.jpg"
  alt: "Raspberry Pi × MQTT × Webhook で現場通知システム"
  relative: false
---

## 導入：現場で起きがちな課題

中小企業の現場で、こんな声をよく聞きます。

- 「バッチ処理が終わったかどうか、毎回サーバーにSSHして確認してる」
- 「倉庫の在庫が減ったら通知してほしいけど、クラウド導入は大げさすぎる」
- 「受付にお客さんが来たら、奥のスタッフに通知したい。でも内線は取れないことが多い」

**共通しているのは「今すぐ知りたいのに、気づくのが遅い」という問題です。**

AWS IoTやAzure IoT Hubは確かに優秀ですが、月額費用、学習コスト、ネットワーク構成の複雑さを考えると、「まずは社内LANで試したい」という現場には重すぎます。

---

## 結論：Pi1台 + MQTT + Webhook + Slackで"即通知"が作れる

この記事で作るシステムは、以下の構成です。

- **Raspberry Pi 1台**（ブローカー兼サブスクライバー）
- **MQTT**（軽量メッセージングプロトコル）
- **Webhook**（HTTPでメッセージを受け取る入口）
- **Slack Incoming Webhook**（通知先）

**外部クラウドは不要。** LAN内で完結し、必要になったら後からAWS IoTへ移行できます。

> **Webhookは"入口"、MQTTは"配線"、処理は"外出し"。** この設計思想を押さえれば、現場通知システムは驚くほどシンプルに作れます。

---

## 全体アーキテクチャ

```mermaid
graph LR
    subgraph 発火源
        A[Mac / PC]
        B[スマホ]
        C[センサー / スクリプト]
    end

    subgraph Raspberry Pi
        D[Mosquitto Broker]
        E[Subscriber Script]
        F[HTTP→MQTT Bridge]
    end

    subgraph 通知先
        G[Slack]
        H[LINE]
        I[メール]
    end

    A -->|MQTT publish| D
    B -->|HTTP POST| F
    C -->|MQTT publish| D
    F -->|publish| D
    D -->|subscribe| E
    E -->|Webhook POST| G
    E -->|Webhook POST| H
    E -->|SMTP| I
```

**ポイント：**
- スマホやブラウザからはHTTP経由で、PiがMQTTに変換する
- Subscriber（処理担当）は通知先ごとに分離できる
- **最初は1台でOK。壊れ始めたら分ければいい。**

---

## 用語を最短で整理

| 用語 | 一言で | よくある誤解 |
|------|--------|-------------|
| **MQTT** | 軽量なPub/Subプロトコル | P2Pではない。必ずBrokerを経由する |
| **Broker** | メッセージの仲介サーバー | MosquittoはBrokerの「実装」の一つ |
| **Pub/Sub** | 発行者と購読者の非同期通信 | 1対1ではなく、1対多も可能 |
| **Webhook** | HTTPでイベントを受け取る仕組み | MQTTとは別物。変換が必要 |
| **Incoming Webhook** | Slack等がPOSTを受け付けるURL | SlackがWebhookを「受ける」側 |
| **systemd** | Linuxのサービス管理機構 | 再起動時に自動起動させるために使う |

### MQTT Pub/Sub の仕組み（図解）

```mermaid
graph TB
    subgraph Publishers["発行者 (Publishers)"]
        Pub1["センサー"]
        Pub2["Mac/PC"]
        Pub3["スマホ"]
    end

    subgraph Broker["MQTTブローカー (Mosquitto)"]
        Topics["トピック管理<br/>sensor/temp<br/>alert/critical<br/>batch/completed"]
    end

    subgraph Subscribers["購読者 (Subscribers)"]
        Sub1["通知スクリプト"]
        Sub2["データ保存"]
        Sub3["ダッシュボード"]
    end

    Pub1 -->|"publish<br/>sensor/temp"| Topics
    Pub2 -->|"publish<br/>alert/critical"| Topics
    Pub3 -->|"publish<br/>batch/completed"| Topics

    Topics -->|"subscribe<br/>sensor/#"| Sub1
    Topics -->|"subscribe<br/>alert/#"| Sub1
    Topics -->|"subscribe<br/>#"| Sub2
    Topics -->|"subscribe<br/>sensor/temp"| Sub3

    style Publishers fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Broker fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Subscribers fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

**重要ポイント：**
- PublisherとSubscriberは**直接通信しない**（必ずBrokerを経由）
- トピックで**メッセージを分類**（`#`はワイルドカード）
- 1つのメッセージを**複数のSubscriberが受信可能**
- SubscriberがオフラインでもPublisherは送信できる（非同期）

### MQTTとSQSの違い（よく聞かれる）

| 項目 | MQTT | Amazon SQS |
|------|------|------------|
| プロトコル | MQTT（TCP） | HTTP(S) |
| モデル | Pub/Sub（ブローカー型） | キュー型（ポーリング） |
| 用途 | IoT、リアルタイム通知 | 非同期タスク処理 |
| セルフホスト | 可能（Mosquitto等） | 不可（AWSサービス） |

**SQSはMQTTではありません。** 似たようなメッセージングでも、設計思想が異なります。

---

## データフローの詳細

```mermaid
sequenceDiagram
    participant Mac as Mac / スマホ
    participant Broker as Mosquitto Broker
    participant Sub as Subscriber Script
    participant Slack as Slack Webhook

    Mac->>Broker: PUBLISH "notify/slack" "倉庫Aの在庫が10個を切りました"
    Broker->>Sub: DELIVER (topic: notify/slack)
    Sub->>Sub: メッセージ整形
    Sub->>Slack: POST /services/xxx {"text": "..."}
    Slack-->>Sub: 200 OK
    Note over Sub: 即200を返すのが鉄則
```

**遅いのはMQTTではなく通知先（Slackなど）。** MQTTは数ミリ秒で届きますが、Slack APIは数百ミリ秒かかることがあります。

---

## 責務分離の設計

```mermaid
graph TB
    subgraph 入口層
        A[HTTP Endpoint]
        B[MQTT Client]
    end

    subgraph メッセージング層
        C[Mosquitto Broker]
    end

    subgraph 処理層
        D[Slack Notifier]
        E[LINE Notifier]
        F[ログ保存]
        G[DB書き込み]
    end

    A --> C
    B --> C
    C --> D
    C --> E
    C --> F
    C --> G

    style C fill:#f9f,stroke:#333,stroke-width:2px
```

**設計原則：**
1. **入口（Webhook/MQTT Client）** は受け取ったら即座にBrokerへ投げる
2. **Broker** は配送に専念（ロジックを持たない）
3. **処理（Subscriber）** は通知先ごとに独立させる

---

## セットアップ全体フロー

実装を始める前に、全体の流れを把握しましょう。

```mermaid
flowchart TD
    Start["開始"] --> Step1["Step 1<br/>Mosquitto Brokerインストール"]
    Step1 --> Step2["Step 2<br/>LAN接続許可設定"]
    Step2 --> Step3["Step 3<br/>Mac/PCから動作確認<br/>(pub/sub)"]
    Step3 --> Check1{動作OK?}

    Check1 -->|No| Debug1["トラブルシューティング<br/>- ファイアウォール確認<br/>- ポート1883開放"]
    Debug1 --> Step3

    Check1 -->|Yes| Step4["Step 4<br/>Slack Incoming Webhook取得"]
    Step4 --> Step5["Step 5<br/>Subscriber実装<br/>(Python/Bash)"]
    Step5 --> Step6["Step 6<br/>Subscriber動作確認"]
    Step6 --> Check2{Slack通知OK?}

    Check2 -->|No| Debug2["トラブルシューティング<br/>- Webhook URL確認<br/>- ネットワーク疎通"]
    Debug2 --> Step6

    Check2 -->|Yes| Step7["Step 7<br/>systemdでサービス化"]
    Step7 --> Step8["Step 8<br/>自動起動設定"]
    Step8 --> Optional["オプション<br/>HTTP→MQTTブリッジ追加"]
    Optional --> Complete["完成！"]

    style Start fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style Complete fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style Debug1 fill:#ffebee,stroke:#f44336,stroke-width:2px
    style Debug2 fill:#ffebee,stroke:#f44336,stroke-width:2px
    style Step3 fill:#fff3e0,stroke:#f57c00
    style Step6 fill:#fff3e0,stroke:#f57c00
```

**所要時間目安：**
- Step 1〜3: 15分
- Step 4〜6: 20分
- Step 7〜8: 10分
- オプション: 15分

**合計: 約60分**（トラブルがなければ）

---

## 最小実装チュートリアル

### 前提環境

- Raspberry Pi OS（Bookworm推奨）
- Python 3.9以上
- ネットワーク接続済み

### Step 1: PiにMosquittoを入れて起動

```bash
# Raspberry Pi で実行
sudo apt update
sudo apt install -y mosquitto mosquitto-clients

# 起動確認
sudo systemctl status mosquitto
```

**期待する出力：**
```
● mosquitto.service - Mosquitto MQTT Broker
     Loaded: loaded (/lib/systemd/system/mosquitto.service; enabled)
     Active: active (running) since ...
```

#### LAN内からの接続を許可する設定

```bash
# 設定ファイルを編集
sudo nano /etc/mosquitto/conf.d/local.conf
```

以下を記述：

```conf
listener 1883 0.0.0.0
allow_anonymous true
```

```bash
# 設定を反映
sudo systemctl restart mosquitto

# ポート確認
ss -tlnp | grep 1883
```

> **セキュリティ注意：** `allow_anonymous true` はLAN内専用の設定です。インターネット公開する場合は認証を必ず設定してください（後述）。

### Step 2: MacからpublishしてPiでsubscribe確認

#### Mac側の準備

```bash
# Mac で実行（Homebrewを使用）
brew install mosquitto
```

#### Topic設計（最小構成）

| Topic | 用途 | 例 |
|-------|------|-----|
| `notify/slack` | Slack通知用 | 汎用通知 |
| `events/warehouse/stock` | 倉庫の在庫イベント | 業務別に分離 |
| `cmd/action` | コマンド発行用 | リモート操作 |

#### 動作確認

**ターミナル1（Pi側でsubscribe）：**

```bash
# Raspberry Pi で実行
mosquitto_sub -h localhost -t "notify/slack" -v
```

**ターミナル2（Mac側でpublish）：**

```bash
# Mac で実行（PiのIPアドレスを指定）
mosquitto_pub -h 192.168.1.100 -t "notify/slack" -m "テストメッセージ"
```

**Pi側の出力：**
```
notify/slack テストメッセージ
```

これでMQTTの疎通確認は完了です。

### Step 3: PiのsubscriberがSlack Incoming Webhookへ通知

#### Slack Incoming Webhookの取得

1. [Slack API](https://api.slack.com/apps) で新規アプリを作成
2. 「Incoming Webhooks」を有効化
3. ワークスペースにインストールしてWebhook URLを取得

#### 環境変数の設定

```bash
# Raspberry Pi で実行
echo 'export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/xxx/yyy/zzz"' >> ~/.bashrc
source ~/.bashrc
```

#### Python版 Subscriber（推奨）

```bash
# 必要なライブラリをインストール
pip install paho-mqtt requests
```

`/home/pi/mqtt_to_slack.py` として保存：

```python
#!/usr/bin/env python3
"""
MQTT → Slack 通知スクリプト
Topic: notify/slack を購読し、Slackへ転送する
"""

import os
import json
import requests
import paho.mqtt.client as mqtt
from datetime import datetime

# 設定
BROKER_HOST = "localhost"
BROKER_PORT = 1883
TOPIC = "notify/slack"
SLACK_WEBHOOK_URL = os.environ.get("SLACK_WEBHOOK_URL")

if not SLACK_WEBHOOK_URL:
    raise ValueError("環境変数 SLACK_WEBHOOK_URL が設定されていません")


def send_to_slack(message: str) -> bool:
    """Slackへメッセージを送信"""
    payload = {
        "text": f"📢 *通知* ({datetime.now().strftime('%H:%M:%S')})\n{message}"
    }
    try:
        response = requests.post(
            SLACK_WEBHOOK_URL,
            json=payload,
            timeout=10
        )
        return response.status_code == 200
    except requests.RequestException as e:
        print(f"Slack送信エラー: {e}")
        return False


def on_connect(client, userdata, flags, rc, properties=None):
    """接続時のコールバック"""
    if rc == 0:
        print(f"Brokerに接続しました: {BROKER_HOST}:{BROKER_PORT}")
        client.subscribe(TOPIC)
        print(f"Topic '{TOPIC}' を購読開始")
    else:
        print(f"接続失敗: rc={rc}")


def on_message(client, userdata, msg):
    """メッセージ受信時のコールバック"""
    message = msg.payload.decode("utf-8")
    print(f"[{msg.topic}] {message}")

    if send_to_slack(message):
        print("  → Slack送信成功")
    else:
        print("  → Slack送信失敗")


def main():
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    client.on_connect = on_connect
    client.on_message = on_message

    client.connect(BROKER_HOST, BROKER_PORT, 60)

    print("MQTT Subscriber 起動中... (Ctrl+C で終了)")
    client.loop_forever()


if __name__ == "__main__":
    main()
```

#### 動作確認

```bash
# Raspberry Pi で実行
python3 /home/pi/mqtt_to_slack.py
```

別ターミナルから：

```bash
mosquitto_pub -h localhost -t "notify/slack" -m "倉庫Aの在庫が10個を切りました"
```

Slackに通知が届けば成功です。

#### Bash + curl版（シンプル版）

Python環境がない場合は、bashスクリプトでも実装できます。

`/home/pi/mqtt_to_slack.sh`:

```bash
#!/bin/bash
# MQTT → Slack 通知（bash版）

TOPIC="notify/slack"
WEBHOOK_URL="${SLACK_WEBHOOK_URL}"

mosquitto_sub -h localhost -t "$TOPIC" | while read -r message; do
    timestamp=$(date '+%H:%M:%S')
    payload=$(jq -n --arg text "📢 *通知* ($timestamp)\n$message" '{text: $text}')
    curl -s -X POST -H 'Content-type: application/json' \
        --data "$payload" "$WEBHOOK_URL" > /dev/null
    echo "[$(date)] Sent: $message"
done
```

```bash
chmod +x /home/pi/mqtt_to_slack.sh
```

---

## スマホから叩く方法

MQTTクライアントアプリをインストールする方法もありますが、**現実的にはHTTP経由が最も手軽**です。

### なぜHTTP→MQTTが現実解なのか

| 方式 | メリット | デメリット |
|------|----------|------------|
| MQTTアプリ | 直接通信 | アプリ必須、設定が面倒 |
| HTTP→MQTT | ブラウザ/ショートカットで完結 | 変換用のエンドポイントが必要 |

**結論：HTTP→MQTT変換をPi上に用意するのが最も汎用的。**

### HTTP→MQTTブリッジの処理フロー

```mermaid
sequenceDiagram
    participant Phone as スマホ/ブラウザ
    participant Bridge as HTTP→MQTTブリッジ<br/>(FastAPI)
    participant Broker as Mosquitto Broker
    participant Sub as Subscriber
    participant Slack as Slack

    Phone->>Bridge: POST /publish<br/>{"topic": "alert/critical",<br/>"message": "在庫切れ"}

    Bridge->>Bridge: リクエスト検証<br/>- topic存在確認<br/>- message長さ確認

    alt 検証OK
        Bridge->>Broker: MQTT publish<br/>topic: alert/critical<br/>payload: "在庫切れ"
        Bridge-->>Phone: 200 OK<br/>{"status": "published"}

        Note over Bridge,Broker: ここでHTTPリクエストは完了<br/>（Webhookは即座に返す）

        Broker->>Sub: メッセージ配信<br/>topic: alert/critical
        Sub->>Sub: Slack Webhook作成
        Sub->>Slack: POST Webhook<br/>{"text": "🚨 在庫切れ"}
        Slack-->>Sub: 200 OK

        Note over Sub,Slack: Subscriberが非同期処理<br/>（ブリッジは関与しない）
    else 検証NG
        Bridge-->>Phone: 400 Bad Request<br/>{"error": "invalid topic"}
    end

    style Bridge fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Broker fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Sub fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

**重要な設計ポイント：**
1. **ブリッジは即座に200を返す**（Slack通知完了を待たない）
2. **検証は最小限**（topic/message存在確認のみ）
3. **重い処理はSubscriberに委譲**（ブリッジは軽量に保つ）
4. **エラーは4xx/5xxで明示**（デバッグしやすく）

### 簡易HTTP→MQTTブリッジ（FastAPI版）

```bash
pip install fastapi uvicorn paho-mqtt
```

`/home/pi/http_to_mqtt.py`:

```python
#!/usr/bin/env python3
"""
HTTP → MQTT ブリッジ
GET/POSTでメッセージを受け取り、MQTTへpublishする
"""

from fastapi import FastAPI, Query
from pydantic import BaseModel
import paho.mqtt.client as mqtt
import uvicorn

app = FastAPI()

# MQTT設定
BROKER_HOST = "localhost"
BROKER_PORT = 1883


class NotifyRequest(BaseModel):
    topic: str = "notify/slack"
    message: str


def publish_mqtt(topic: str, message: str):
    """MQTTへpublish"""
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    client.connect(BROKER_HOST, BROKER_PORT, 60)
    client.publish(topic, message)
    client.disconnect()


@app.get("/notify")
async def notify_get(
    message: str = Query(..., description="通知メッセージ"),
    topic: str = Query("notify/slack", description="MQTTトピック")
):
    """GETで通知（ブラウザ/ショートカット向け）"""
    publish_mqtt(topic, message)
    return {"status": "ok", "topic": topic, "message": message}


@app.post("/notify")
async def notify_post(req: NotifyRequest):
    """POSTで通知（アプリ連携向け）"""
    publish_mqtt(req.topic, req.message)
    return {"status": "ok", "topic": req.topic, "message": req.message}


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8080)
```

#### 起動

```bash
python3 /home/pi/http_to_mqtt.py
```

#### 使用例

**ブラウザから：**
```
http://192.168.1.100:8080/notify?message=お客様が来店しました
```

**iPhoneショートカットから：**
1. 「ショートカット」アプリを開く
2. 「URLの内容を取得」アクションを追加
3. URL: `http://192.168.1.100:8080/notify?message=帰宅しました`
4. ホーム画面に追加

**curlから：**
```bash
# GET
curl "http://192.168.1.100:8080/notify?message=テスト通知"

# POST
curl -X POST http://192.168.1.100:8080/notify \
  -H "Content-Type: application/json" \
  -d '{"topic": "notify/slack", "message": "在庫補充完了"}'
```

---

## systemdで常駐化

### systemd設定の全体像

```mermaid
graph TB
    subgraph Boot["Pi起動時"]
        Start["システム起動"]
        Network["ネットワーク起動<br/>(network.target)"]
        Mosquitto["Mosquitto起動<br/>(mosquitto.service)"]
    end

    subgraph Services["MQTTサービス群"]
        Subscriber["Subscriber起動<br/>(mqtt-to-slack.service)"]
        Bridge["HTTP→MQTTブリッジ起動<br/>(http-to-mqtt.service)"]
    end

    subgraph Monitor["監視・自動再起動"]
        Check{サービス<br/>異常終了?}
        Restart["10秒後に自動再起動<br/>(RestartSec=10)"]
    end

    Start --> Network
    Network --> Mosquitto
    Mosquitto --> Subscriber
    Mosquitto --> Bridge

    Subscriber --> Check
    Bridge --> Check
    Check -->|Yes| Restart
    Restart --> Subscriber
    Restart --> Bridge
    Check -->|No| Running["正常稼働"]

    style Boot fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Services fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style Monitor fill:#fff3e0,stroke:#f57c00,stroke-width:2px
```

**systemdの利点：**
- **自動起動**：Pi再起動時に自動で全サービス起動
- **依存関係管理**：Mosquitto起動後にSubscriberを起動
- **自動再起動**：異常終了時に自動復旧
- **ログ管理**：journalctlで一元管理

### Subscriberのサービス化

`/etc/systemd/system/mqtt-to-slack.service`:

```ini
[Unit]
Description=MQTT to Slack Notifier
After=network.target mosquitto.service

[Service]
Type=simple
User=pi
Environment="SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx/yyy/zzz"
ExecStart=/usr/bin/python3 /home/pi/mqtt_to_slack.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# サービス登録・起動
sudo systemctl daemon-reload
sudo systemctl enable mqtt-to-slack
sudo systemctl start mqtt-to-slack

# 状態確認
sudo systemctl status mqtt-to-slack
```

### HTTP→MQTTブリッジのサービス化

`/etc/systemd/system/http-to-mqtt.service`:

```ini
[Unit]
Description=HTTP to MQTT Bridge
After=network.target mosquitto.service

[Service]
Type=simple
User=pi
ExecStart=/usr/bin/python3 /home/pi/http_to_mqtt.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable http-to-mqtt
sudo systemctl start http-to-mqtt
```

### ログ閲覧

```bash
# リアルタイムログ
sudo journalctl -u mqtt-to-slack -f

# 過去ログ（直近100行）
sudo journalctl -u mqtt-to-slack -n 100

# 今日のログ
sudo journalctl -u mqtt-to-slack --since today
```

### 再起動コマンド

```bash
# 設定変更後
sudo systemctl restart mqtt-to-slack

# 停止
sudo systemctl stop mqtt-to-slack
```

---

## 壊れやすいポイントと対策

### トラブルシューティングフローチャート

```mermaid
flowchart TD
    Problem["通知が届かない"] --> Check1{Brokerは<br/>起動している?}

    Check1 -->|No| Fix1["sudo systemctl start mosquitto<br/>sudo systemctl enable mosquitto"]
    Fix1 --> Solved1["解決"]

    Check1 -->|Yes| Check2{Publishは<br/>成功している?}

    Check2 -->|No| Check3{ファイアウォールで<br/>1883ポートが開いている?}
    Check3 -->|No| Fix2["sudo ufw allow 1883<br/>または<br/>mosquitto.confを確認"]
    Fix2 --> Solved1

    Check3 -->|Yes| Check4{認証エラーが<br/>出ている?}
    Check4 -->|Yes| Fix3["mosquitto.conf<br/>allow_anonymous true確認"]
    Fix3 --> Solved1

    Check2 -->|Yes| Check5{Subscriberは<br/>起動している?}

    Check5 -->|No| Fix4["sudo systemctl start<br/>mqtt-to-slack"]
    Fix4 --> Solved1

    Check5 -->|Yes| Check6{Subscriberログに<br/>エラーがある?}

    Check6 -->|Yes| Check7{Slack Webhook<br/>URLは正しい?}
    Check7 -->|No| Fix5["環境変数を確認<br/>SLACK_WEBHOOK_URL"]
    Fix5 --> Solved1

    Check7 -->|Yes| Check8{ネットワークは<br/>疎通している?}
    Check8 -->|No| Fix6["curl https://hooks.slack.com<br/>で疎通確認"]
    Fix6 --> Solved1

    Check8 -->|Yes| Fix7["QoS設定確認<br/>重複メッセージの可能性"]
    Fix7 --> Solved1

    Check6 -->|No| Fix8["topicが一致しているか確認<br/>pub: alert/critical<br/>sub: alert/#"]
    Fix8 --> Solved1

    style Problem fill:#ffebee,stroke:#f44336,stroke-width:2px
    style Solved1 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style Fix1 fill:#fff3e0,stroke:#f57c00
    style Fix2 fill:#fff3e0,stroke:#f57c00
    style Fix3 fill:#fff3e0,stroke:#f57c00
    style Fix4 fill:#fff3e0,stroke:#f57c00
    style Fix5 fill:#fff3e0,stroke:#f57c00
    style Fix6 fill:#fff3e0,stroke:#f57c00
    style Fix7 fill:#fff3e0,stroke:#f57c00
    style Fix8 fill:#fff3e0,stroke:#f57c00
```

**よくある問題トップ3：**
1. **Mosquitto未起動**（`sudo systemctl status mosquitto`で確認）
2. **topic名の不一致**（`alert/critical` vs `alert/#`）
3. **Slack Webhook URL間違い**（環境変数を再確認）

### 1. 処理が重いとSubscriberが詰まる

**問題：** Slack APIが遅い、またはタイムアウトするとメッセージが滞留

**対策：**
```python
# 非同期処理にする（簡易版）
import threading

def on_message(client, userdata, msg):
    message = msg.payload.decode("utf-8")
    # バックグラウンドで送信
    threading.Thread(target=send_to_slack, args=(message,)).start()
```

### 2. Webhookエンドポイントが即200を返さない

**問題：** 外部からのHTTPリクエストがタイムアウトする

**対策：** Webhookは受け取ったら即座に200を返し、処理はバックグラウンドで実行

```python
@app.post("/notify")
async def notify_post(req: NotifyRequest, background_tasks: BackgroundTasks):
    """即200を返し、MQTT送信はバックグラウンドで"""
    background_tasks.add_task(publish_mqtt, req.topic, req.message)
    return {"status": "accepted"}
```

### 3. QoSと重複メッセージ

| QoS | 意味 | 用途 |
|-----|------|------|
| 0 | 最大1回（届かないかも） | ログ、センサー値 |
| 1 | 最低1回（重複あり） | 通知（推奨） |
| 2 | 必ず1回（オーバーヘッド大） | 決済等（MQTTでは稀） |

**対策：** 通知システムはQoS 1で十分。冪等性を意識した設計にする。

### 4. Brokerがダウンしたら

**対策：**
- systemdの `Restart=always` で自動復旧
- 監視スクリプトを別途用意

```bash
# 簡易監視（cron登録用）
mosquitto_pub -h localhost -t "health/check" -m "ping" || \
  sudo systemctl restart mosquitto
```

---

## セキュリティ注意点

### セキュリティレイヤー構成

```mermaid
graph TB
    subgraph Internet["インターネット"]
        Attacker["外部からの攻撃"]
    end

    subgraph Router["ルーター / ファイアウォール"]
        FW["ファイアウォール<br/>1883/8080ポート<br/>❌ 外部公開しない"]
    end

    subgraph LAN["社内LAN（推奨構成）"]
        subgraph Auth["認証レイヤー"]
            MQTTAuth["MQTT認証<br/>mosquitto_passwd"]
            HTTPAuth["HTTP Basic Auth<br/>（オプション）"]
        end

        subgraph App["アプリケーションレイヤー"]
            Broker["Mosquitto Broker<br/>0.0.0.0:1883"]
            Bridge["HTTP→MQTTブリッジ<br/>0.0.0.0:8080"]
        end

        subgraph Access["アクセス制御"]
            AllowList["許可リスト<br/>192.168.1.0/24のみ"]
        end
    end

    subgraph Public["公開する場合（非推奨）"]
        TLS["TLS/SSL暗号化<br/>8883ポート"]
        RateLimit["レート制限<br/>Fail2ban"]
        VPN["VPN経由アクセス<br/>（強く推奨）"]
    end

    Attacker -.->|ブロック| FW
    FW --> Router
    Router --> AllowList
    AllowList --> MQTTAuth
    AllowList --> HTTPAuth
    MQTTAuth --> Broker
    HTTPAuth --> Bridge

    style Internet fill:#ffebee,stroke:#f44336,stroke-width:2px
    style FW fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style LAN fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style Auth fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Public fill:#fff3e0,stroke:#ff9800,stroke-width:2px
```

**セキュリティレベル別推奨構成：**

| レベル | 構成 | 用途 |
|--------|------|------|
| **Level 1<br/>（最小）** | LAN限定 + `allow_anonymous true` | 社内PoC、個人利用 |
| **Level 2<br/>（推奨）** | LAN限定 + MQTT認証 + IPフィルタ | 小規模運用 |
| **Level 3<br/>（堅牢）** | VPN + TLS + 認証 + レート制限 | 外部アクセスが必要な場合 |

### LAN限定運用（推奨）

- ルーターで1883/8080ポートを外部公開しない
- `allow_anonymous true` は社内LAN限定

### 認証を追加する場合

```bash
# パスワードファイル作成
sudo mosquitto_passwd -c /etc/mosquitto/passwd mqttuser
```

`/etc/mosquitto/conf.d/local.conf`:
```conf
listener 1883 0.0.0.0
allow_anonymous false
password_file /etc/mosquitto/passwd
```

```bash
sudo systemctl restart mosquitto
```

クライアント側：
```bash
mosquitto_pub -h 192.168.1.100 -u mqttuser -P password -t "notify/slack" -m "test"
```

### Webhook URLの保護

- Slack Webhook URLは漏洩すると誰でも投稿できる
- 環境変数で管理し、GitHubに絶対にpushしない
- 定期的にURLを再生成する運用も検討

### 公開する場合（非推奨だが必要なら）

- TLS/SSL必須（Let's Encrypt + nginx リバースプロキシ）
- API Key認証を追加
- レートリミットを設定

---

## 実践ユースケース集

この構成は以下のような現場で実際に活用できます。

### ユースケース① 倉庫在庫管理

```mermaid
graph LR
    subgraph Warehouse["倉庫"]
        Sensor["在庫センサー<br/>（しきい値検知）"]
        Pi["Raspberry Pi<br/>+ MQTT Broker"]
    end

    subgraph Office["事務所"]
        Manager["在庫管理者<br/>（Slack受信）"]
    end

    Sensor -->|"MQTT publish<br/>topic: stock/low<br/>payload: A棚在庫5個"| Pi
    Pi -->|"Subscriber→Webhook"| Manager

    style Warehouse fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Office fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

**実装例：**
- センサー：Arduinoでカウント → MQTTで送信
- 通知：「🚨 A棚の在庫が5個まで減少しました」

### ユースケース② バッチ処理完了通知

```mermaid
sequenceDiagram
    participant Batch as バッチサーバー
    participant Pi as Raspberry Pi<br/>(MQTT Broker)
    participant Slack as 担当者<br/>(Slack)

    Note over Batch: 深夜2時: バッチ開始
    Batch->>Batch: データ処理実行

    alt 成功
        Batch->>Pi: MQTT publish<br/>topic: batch/success
        Pi->>Slack: ✅ バッチ処理完了<br/>処理時間: 45分
    else 失敗
        Batch->>Pi: MQTT publish<br/>topic: batch/error
        Pi->>Slack: ❌ バッチ処理失敗<br/>エラーログ確認必要
    end

    Note over Slack: 朝、出社時に確認
```

**実装例：**
```bash
# バッチスクリプトの最後に
if [ $? -eq 0 ]; then
  mosquitto_pub -h 192.168.1.100 -t "batch/success" -m "処理完了"
else
  mosquitto_pub -h 192.168.1.100 -t "batch/error" -m "処理失敗"
fi
```

### ユースケース③ 受付呼び出しシステム

```mermaid
graph TB
    subgraph Reception["受付"]
        Button["呼び出しボタン<br/>（スマホショートカット）"]
    end

    subgraph Pi["Raspberry Pi"]
        Bridge["HTTP→MQTTブリッジ"]
        Broker["MQTT Broker"]
        Sub["Subscriber"]
    end

    subgraph Backoffice["事務所・倉庫"]
        Staff1["スタッフA<br/>（Slack）"]
        Staff2["スタッフB<br/>（Slack）"]
    end

    Button -->|"HTTP POST<br/>http://pi.local:8080/notify"| Bridge
    Bridge --> Broker
    Broker --> Sub
    Sub --> Staff1
    Sub --> Staff2

    style Reception fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style Pi fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style Backoffice fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

**実装例：**
- iPhoneショートカット：ボタン1タップでHTTP POST
- 通知：「📞 受付にお客様がいらっしゃいました」

---

## まとめ

この記事では、**Raspberry Pi 1台で構築できる現場通知システム**を解説しました。

### 構成のポイント

1. **Mosquitto**をブローカーとしてLAN内に設置
2. **MQTT**でメッセージを配信（軽量・高速）
3. **Webhook**でHTTPからの入力を受け付け
4. **systemd**で常駐化・自動復旧

### この設計の強み

- **AWSなしで動く**：月額費用ゼロ、LAN完結
- **拡張性がある**：通知先を増やすのはSubscriberを追加するだけ
- **移行パスがある**：必要になればAWS IoTへスムーズに移行可能

> **最初は1台でOK。壊れ始めたら分ければいい。**

小さく始めて、効果を確認しながら拡大していく。それが現場通知システム導入の正解です。

