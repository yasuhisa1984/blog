---
title: "WebSocket完全理解：リアルタイム通信の設計と実装"
date: 2025-12-14
draft: false
categories: ["システム設計"]
tags: ["WebSocket", "リアルタイム", "システム設計", "実務", "Socket.IO"]
description: "HTTPの限界からWebSocketが必要な理由、ポーリングとの違い、実装パターン、スケールアウト時の課題と解決策まで。チャット、通知、株価更新など実務でのユースケースを交えて解説。"
cover:
  image: "images/covers/websocket-realtime.jpg"
  alt: "WebSocket リアルタイム通信"
  relative: false
---

## はじめに

「5秒ごとにAPIを叩いて更新を確認する」

これ、いつまで続けますか？

ポーリングは動く。でも、
- サーバーに無駄な負荷がかかる
- リアルタイム性が犠牲になる
- バッテリーを消耗する

**本当にリアルタイムが必要なら、WebSocketを使え。**

この記事では、WebSocketの本質から、実務での設計・実装パターン、スケールアウト時の課題と解決策までを解説する。

---

## HTTPの限界：なぜWebSocketが必要か

### HTTPはリクエスト・レスポンス型

```
クライアント → リクエスト → サーバー
クライアント ← レスポンス ← サーバー
接続終了
```

HTTPでは、**クライアントからしかリクエストを送れない**。

サーバーが「新しいメッセージが来たよ」と伝えたくても、クライアントが聞きに来るまで待つしかない。

### ポーリングの問題

```javascript
// ショートポーリング: 定期的にAPIを叩く
setInterval(async () => {
  const messages = await fetch('/api/messages');
  // 更新があっても、なくても、5秒ごとにリクエスト
}, 5000);
```

**問題点**:

| 問題 | 説明 |
|------|------|
| 無駄なリクエスト | 更新がなくても叩き続ける |
| 遅延 | 最大5秒の遅れ（インターバル依存） |
| サーバー負荷 | ユーザー数 × リクエスト/秒 |
| バッテリー消耗 | モバイルで顕著 |

### ロングポーリング

```javascript
// ロングポーリング: サーバーが更新まで待機
async function longPoll() {
  const response = await fetch('/api/messages/wait');  // サーバーは更新まで応答しない
  handleMessages(response);
  longPoll();  // 再接続
}
```

**改善点**:
- 更新がない時のリクエストが減る
- リアルタイム性が向上

**残る問題**:
- 毎回HTTP接続を張り直す
- HTTPヘッダーのオーバーヘッド
- サーバー側のコネクション管理が複雑

### WebSocketの解決策

```
クライアント ← → サーバー
（常時接続、双方向）
```

一度接続すれば、**双方向でいつでもデータを送れる**。

---

## WebSocketの仕組み

### ハンドシェイク

WebSocketは、HTTPでハンドシェイクしてから、プロトコルを切り替える。

```http
# クライアント → サーバー
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

# サーバー → クライアント
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**101 Switching Protocols**の後、HTTP接続はWebSocket接続に昇格する。

### WebSocketフレーム

ハンドシェイク後は、**軽量なフレーム形式**でデータをやり取りする。

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

**HTTPヘッダー（数百バイト）に比べて、わずか2〜14バイト**のオーバーヘッド。

---

## WebSocket vs SSE vs ポーリング

### 比較表

| 方式 | 方向 | リアルタイム | 複雑さ | ユースケース |
|------|------|------------|--------|-------------|
| ショートポーリング | クライアント→サーバー | ❌ 低い | ✅ 簡単 | 更新頻度が低い |
| ロングポーリング | クライアント→サーバー | ⚪ 中程度 | ⚪ 普通 | レガシー対応 |
| SSE | サーバー→クライアント | ✅ 高い | ✅ 簡単 | 通知、フィード |
| WebSocket | 双方向 | ✅ 高い | ❌ 複雑 | チャット、ゲーム |

### SSE（Server-Sent Events）

サーバーからクライアントへの**一方向**ストリーム。

```javascript
// クライアント
const eventSource = new EventSource('/api/notifications');

eventSource.onmessage = (event) => {
  console.log('New notification:', event.data);
};
```

```python
# サーバー（Python/Flask）
@app.route('/api/notifications')
def notifications():
    def generate():
        while True:
            notification = get_next_notification()
            yield f"data: {json.dumps(notification)}\n\n"

    return Response(generate(), mimetype='text/event-stream')
```

**SSEが適しているケース**:
- 株価更新
- ニュースフィード
- プッシュ通知
- ログのリアルタイム表示

**SSEの利点**:
- 実装が簡単
- 自動再接続
- HTTPのまま（プロキシを通りやすい）

### 使い分けの指針

```
サーバー → クライアントだけ？ → SSE
双方向が必要？ → WebSocket
リアルタイム不要？ → ポーリング
```

---

## 【実装】基本的なWebSocketサーバー

### Node.js（ws）

```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

// 接続管理
const clients = new Set();

wss.on('connection', (ws) => {
  console.log('Client connected');
  clients.add(ws);

  // メッセージ受信
  ws.on('message', (data) => {
    const message = JSON.parse(data);
    console.log('Received:', message);

    // 全クライアントにブロードキャスト
    broadcast(message);
  });

  // 切断
  ws.on('close', () => {
    console.log('Client disconnected');
    clients.delete(ws);
  });

  // エラー
  ws.on('error', (error) => {
    console.error('WebSocket error:', error);
  });
});

function broadcast(message) {
  const data = JSON.stringify(message);
  clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(data);
    }
  });
}
```

### Python（websockets）

```python
import asyncio
import websockets
import json

clients = set()

async def handler(websocket, path):
    clients.add(websocket)
    try:
        async for message in websocket:
            data = json.loads(message)
            print(f"Received: {data}")

            # ブロードキャスト
            await broadcast(data)
    finally:
        clients.remove(websocket)

async def broadcast(message):
    data = json.dumps(message)
    await asyncio.gather(
        *[client.send(data) for client in clients if client.open]
    )

start_server = websockets.serve(handler, "localhost", 8080)

asyncio.get_event_loop().run_until_complete(start_server)
asyncio.get_event_loop().run_forever()
```

### クライアント側（JavaScript）

```javascript
const ws = new WebSocket('ws://localhost:8080');

// 接続成功
ws.onopen = () => {
  console.log('Connected');
  ws.send(JSON.stringify({ type: 'join', user: 'yamada' }));
};

// メッセージ受信
ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};

// 切断
ws.onclose = (event) => {
  console.log('Disconnected:', event.code, event.reason);
};

// エラー
ws.onerror = (error) => {
  console.error('Error:', error);
};

// メッセージ送信
function sendMessage(text) {
  ws.send(JSON.stringify({ type: 'message', text }));
}
```

---

## 【実装】Socket.IOを使った実装

Socket.IOは、WebSocketをより使いやすくしたライブラリ。

### 特徴

- **フォールバック**: WebSocketが使えない環境でもポーリングにフォールバック
- **自動再接続**: 切断時に自動で再接続
- **ルーム機能**: グループへのブロードキャストが簡単
- **名前空間**: 機能ごとに接続を分離

### サーバー（Node.js）

```javascript
const { Server } = require('socket.io');
const io = new Server(3000, {
  cors: {
    origin: "http://localhost:8080",
    methods: ["GET", "POST"]
  }
});

io.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  // ルームに参加
  socket.on('join_room', (room) => {
    socket.join(room);
    console.log(`${socket.id} joined ${room}`);

    // ルームの他のメンバーに通知
    socket.to(room).emit('user_joined', { userId: socket.id });
  });

  // メッセージ送信
  socket.on('send_message', (data) => {
    const { room, message } = data;

    // ルーム内にブロードキャスト（送信者以外）
    socket.to(room).emit('receive_message', {
      userId: socket.id,
      message,
      timestamp: new Date().toISOString()
    });
  });

  // タイピング中
  socket.on('typing', (room) => {
    socket.to(room).emit('user_typing', { userId: socket.id });
  });

  // 切断
  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});
```

### クライアント

```javascript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000');

// 接続
socket.on('connect', () => {
  console.log('Connected:', socket.id);
  socket.emit('join_room', 'room-1');
});

// メッセージ受信
socket.on('receive_message', (data) => {
  console.log('Message:', data);
  displayMessage(data);
});

// ユーザー参加
socket.on('user_joined', (data) => {
  console.log('User joined:', data.userId);
});

// タイピング中表示
socket.on('user_typing', (data) => {
  showTypingIndicator(data.userId);
});

// メッセージ送信
function sendMessage(message) {
  socket.emit('send_message', {
    room: 'room-1',
    message
  });
}

// タイピング通知
function notifyTyping() {
  socket.emit('typing', 'room-1');
}
```

---

## 【実務】メッセージプロトコルの設計

### メッセージ形式

```javascript
// 基本形式
{
  "type": "message_type",
  "payload": { ... },
  "timestamp": "2024-12-14T10:30:00Z",
  "messageId": "uuid"
}
```

### イベント型の設計

```javascript
// チャットアプリの例
const MessageTypes = {
  // クライアント → サーバー
  JOIN_ROOM: 'join_room',
  LEAVE_ROOM: 'leave_room',
  SEND_MESSAGE: 'send_message',
  TYPING_START: 'typing_start',
  TYPING_END: 'typing_end',
  READ_MESSAGES: 'read_messages',

  // サーバー → クライアント
  USER_JOINED: 'user_joined',
  USER_LEFT: 'user_left',
  NEW_MESSAGE: 'new_message',
  USER_TYPING: 'user_typing',
  MESSAGES_READ: 'messages_read',
  ERROR: 'error',
};

// 使用例
{
  "type": "send_message",
  "payload": {
    "roomId": "room-123",
    "content": "Hello!",
    "replyTo": null
  },
  "messageId": "msg-456"
}
```

### ACK（確認応答）パターン

```javascript
// クライアント側: メッセージ送信と確認
function sendMessageWithAck(content) {
  const messageId = generateUUID();

  // タイムアウト付きで確認を待つ
  const timeout = setTimeout(() => {
    console.error('Message not acknowledged:', messageId);
    retryOrFail(messageId);
  }, 5000);

  socket.emit('send_message', { messageId, content });

  socket.once(`ack:${messageId}`, (response) => {
    clearTimeout(timeout);
    console.log('Message acknowledged:', response);
  });
}

// サーバー側: ACKを返す
socket.on('send_message', async (data) => {
  try {
    const saved = await saveMessage(data);
    socket.emit(`ack:${data.messageId}`, {
      success: true,
      savedId: saved.id
    });

    // 他のユーザーに配信
    socket.to(data.roomId).emit('new_message', saved);
  } catch (error) {
    socket.emit(`ack:${data.messageId}`, {
      success: false,
      error: error.message
    });
  }
});
```

---

## 【実務】再接続とハートビート

### 自動再接続

```javascript
class ReconnectingWebSocket {
  constructor(url, options = {}) {
    this.url = url;
    this.reconnectInterval = options.reconnectInterval || 1000;
    this.maxReconnectInterval = options.maxReconnectInterval || 30000;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = options.maxReconnectAttempts || 10;

    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectAttempts = 0;
      this.reconnectInterval = 1000;
      this.startHeartbeat();
    };

    this.ws.onclose = (event) => {
      console.log('Disconnected:', event.code);
      this.stopHeartbeat();

      if (event.code !== 1000) {  // 正常終了以外
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = (error) => {
      console.error('Error:', error);
    };
  }

  scheduleReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnect attempts reached');
      return;
    }

    this.reconnectAttempts++;

    // 指数バックオフ + ジッター
    const delay = Math.min(
      this.reconnectInterval * Math.pow(2, this.reconnectAttempts - 1),
      this.maxReconnectInterval
    ) * (0.5 + Math.random() * 0.5);

    console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);

    setTimeout(() => this.connect(), delay);
  }

  // ハートビート（後述）
  startHeartbeat() {
    this.heartbeatInterval = setInterval(() => {
      if (this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({ type: 'ping' }));
      }
    }, 30000);
  }

  stopHeartbeat() {
    clearInterval(this.heartbeatInterval);
  }
}
```

### ハートビート（死活監視）

```javascript
// サーバー側
const HEARTBEAT_INTERVAL = 30000;
const HEARTBEAT_TIMEOUT = 10000;

wss.on('connection', (ws) => {
  ws.isAlive = true;

  ws.on('pong', () => {
    ws.isAlive = true;
  });

  ws.on('message', (data) => {
    const message = JSON.parse(data);
    if (message.type === 'ping') {
      ws.send(JSON.stringify({ type: 'pong' }));
      return;
    }
    // 通常のメッセージ処理
  });
});

// 定期的に死活確認
setInterval(() => {
  wss.clients.forEach((ws) => {
    if (ws.isAlive === false) {
      console.log('Client timed out, terminating');
      return ws.terminate();
    }

    ws.isAlive = false;
    ws.ping();
  });
}, HEARTBEAT_INTERVAL);
```

---

## 【実務】スケールアウトの課題

### 問題：サーバーが複数になると...

```
        ┌─────────────────────────────────────┐
        │          Load Balancer              │
        └──────┬──────────────┬───────────────┘
               │              │
        ┌──────▼────┐  ┌──────▼────┐
        │ Server A  │  │ Server B  │
        │ (User 1)  │  │ (User 2)  │
        └───────────┘  └───────────┘

User 1 が User 2 にメッセージを送りたい
→ Server A は User 2 の接続を知らない
→ メッセージが届かない！
```

### 解決策1: Redis Pub/Sub

```javascript
const Redis = require('ioredis');
const pub = new Redis();
const sub = new Redis();

// サーバー間でメッセージを共有
sub.subscribe('chat_messages');

sub.on('message', (channel, message) => {
  const data = JSON.parse(message);

  // このサーバーに接続しているクライアントにのみ送信
  broadcastToLocalClients(data);
});

// メッセージ送信時
function sendMessage(roomId, message) {
  // Redisに発行 → 全サーバーが受信
  pub.publish('chat_messages', JSON.stringify({
    roomId,
    message,
    timestamp: Date.now()
  }));
}
```

```
        ┌─────────────────────────────────────┐
        │          Load Balancer              │
        └──────┬──────────────┬───────────────┘
               │              │
        ┌──────▼────┐  ┌──────▼────┐
        │ Server A  │  │ Server B  │
        │ (User 1)  │  │ (User 2)  │
        └─────┬─────┘  └─────┬─────┘
              │              │
              └──────┬───────┘
                     │
              ┌──────▼──────┐
              │   Redis     │
              │  Pub/Sub    │
              └─────────────┘

User 1 → Server A → Redis → Server B → User 2
```

### 解決策2: Socket.IO + Redis Adapter

```javascript
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const io = new Server(3000);

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
  io.adapter(createAdapter(pubClient, subClient));

  // これだけで、複数サーバー間でルームやブロードキャストが動く
  io.on('connection', (socket) => {
    socket.on('join_room', (room) => {
      socket.join(room);  // 複数サーバーでも動く
    });

    socket.on('send_message', (data) => {
      io.to(data.room).emit('new_message', data);  // 全サーバーに配信
    });
  });
});
```

### 解決策3: Sticky Session

同じユーザーを常に同じサーバーに接続させる。

```nginx
# nginx.conf
upstream websocket_servers {
    ip_hash;  # IPベースでスティッキー
    server server1:3000;
    server server2:3000;
}

server {
    location /socket.io/ {
        proxy_pass http://websocket_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

**注意**: サーバーがダウンすると再接続時に別サーバーになる可能性あり。

---

## 【実務】認証

### 接続時の認証

```javascript
// クライアント
const socket = io('http://localhost:3000', {
  auth: {
    token: 'jwt-token-here'
  }
});

// サーバー
io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  try {
    const user = jwt.verify(token, SECRET_KEY);
    socket.user = user;
    next();
  } catch (error) {
    next(new Error('Authentication failed'));
  }
});

io.on('connection', (socket) => {
  console.log('Authenticated user:', socket.user.id);
});
```

### ルームへのアクセス制御

```javascript
socket.on('join_room', async (roomId) => {
  // ユーザーがこのルームにアクセスする権限があるか確認
  const hasAccess = await checkRoomAccess(socket.user.id, roomId);

  if (!hasAccess) {
    socket.emit('error', { message: 'Access denied' });
    return;
  }

  socket.join(roomId);
  socket.emit('joined_room', { roomId });
});
```

---

## 【実務】よくあるユースケースと実装

### 1. チャットアプリ

```javascript
// ルーム管理
const rooms = new Map();

socket.on('join_room', (roomId) => {
  socket.join(roomId);

  // オンラインユーザーを通知
  const users = getRoomUsers(roomId);
  io.to(roomId).emit('room_users', users);
});

socket.on('send_message', async (data) => {
  const message = await saveMessage({
    roomId: data.roomId,
    userId: socket.user.id,
    content: data.content,
    timestamp: new Date()
  });

  io.to(data.roomId).emit('new_message', message);
});

socket.on('typing', (roomId) => {
  socket.to(roomId).emit('user_typing', {
    userId: socket.user.id,
    userName: socket.user.name
  });
});
```

### 2. 通知システム

```javascript
// ユーザーごとの個別ルームに参加
socket.on('authenticate', (userId) => {
  socket.join(`user:${userId}`);
});

// 特定ユーザーに通知を送る
async function sendNotification(userId, notification) {
  // DBに保存
  await saveNotification(userId, notification);

  // リアルタイム配信
  io.to(`user:${userId}`).emit('notification', notification);
}

// 使用例
await sendNotification(123, {
  type: 'new_message',
  title: '新しいメッセージがあります',
  body: '山田さんからメッセージが届きました',
  url: '/messages/456'
});
```

### 3. リアルタイムダッシュボード

```javascript
// サーバー側: 定期的にデータを配信
setInterval(async () => {
  const metrics = await getSystemMetrics();
  io.to('dashboard').emit('metrics_update', metrics);
}, 1000);

// クライアント側
socket.emit('subscribe', 'dashboard');

socket.on('metrics_update', (metrics) => {
  updateChart(metrics.cpu);
  updateChart(metrics.memory);
  updateChart(metrics.requests);
});
```

### 4. リアルタイム共同編集

```javascript
// Operational Transformation（OT）の簡易版
socket.on('document_change', (data) => {
  const { documentId, changes, version } = data;

  // 変更を他のユーザーにブロードキャスト
  socket.to(`document:${documentId}`).emit('remote_change', {
    userId: socket.user.id,
    changes,
    version
  });
});

// カーソル位置の共有
socket.on('cursor_move', (data) => {
  socket.to(`document:${data.documentId}`).emit('remote_cursor', {
    userId: socket.user.id,
    position: data.position
  });
});
```

### 5. オンラインゲーム（位置同期）

```javascript
// 60FPSで位置を同期
const TICK_RATE = 1000 / 60;

setInterval(() => {
  rooms.forEach((room, roomId) => {
    const gameState = {
      players: Array.from(room.players.values()).map(p => ({
        id: p.id,
        x: p.x,
        y: p.y,
        rotation: p.rotation
      })),
      timestamp: Date.now()
    };

    io.to(roomId).emit('game_state', gameState);
  });
}, TICK_RATE);

// クライアントからの入力
socket.on('player_input', (input) => {
  const player = getPlayer(socket.user.id);
  player.applyInput(input);
});
```

---

## 【実務】パフォーマンス最適化

### 1. メッセージの圧縮

```javascript
const { Server } = require('socket.io');

const io = new Server(3000, {
  perMessageDeflate: {
    threshold: 1024,  // 1KB以上のメッセージを圧縮
    zlibDeflateOptions: {
      chunkSize: 16 * 1024
    }
  }
});
```

### 2. バイナリプロトコル

```javascript
// JSONの代わりにMessagePackを使用
const msgpack = require('msgpack-lite');

// 送信
socket.send(msgpack.encode({ type: 'update', data: largeData }));

// 受信
socket.on('message', (buffer) => {
  const data = msgpack.decode(buffer);
});
```

### 3. メッセージのバッチ処理

```javascript
// 高頻度の更新をバッチ化
class MessageBatcher {
  constructor(socket, interval = 50) {
    this.socket = socket;
    this.queue = [];
    this.interval = interval;

    setInterval(() => this.flush(), interval);
  }

  add(message) {
    this.queue.push(message);
  }

  flush() {
    if (this.queue.length === 0) return;

    this.socket.emit('batch', this.queue);
    this.queue = [];
  }
}
```

### 4. 接続数の制限

```javascript
const MAX_CONNECTIONS_PER_IP = 10;
const connectionCounts = new Map();

io.use((socket, next) => {
  const ip = socket.handshake.address;
  const count = connectionCounts.get(ip) || 0;

  if (count >= MAX_CONNECTIONS_PER_IP) {
    return next(new Error('Too many connections'));
  }

  connectionCounts.set(ip, count + 1);

  socket.on('disconnect', () => {
    connectionCounts.set(ip, connectionCounts.get(ip) - 1);
  });

  next();
});
```

---

## WebSocket実装チェックリスト

### 設計

- [ ] リアルタイムが本当に必要か（SSEで十分では？）
- [ ] メッセージプロトコルは定義されているか
- [ ] 認証・認可は考慮されているか
- [ ] スケールアウト時の設計はあるか

### 実装

- [ ] 再接続ロジックは実装されているか
- [ ] ハートビートは実装されているか
- [ ] エラーハンドリングは適切か
- [ ] メッセージのACKは必要か

### 運用

- [ ] 接続数の監視は設定されているか
- [ ] メッセージのログは取れているか
- [ ] 異常切断の検知はできるか

---

## まとめ

WebSocketの本質は、**常時接続による双方向通信**だ。

- **HTTP**: リクエスト・レスポンス（クライアントが主導）
- **WebSocket**: 双方向（サーバーからも送れる）
- **SSE**: サーバー→クライアント（シンプルで十分な場合）

使い分け:
- 双方向が必要 → WebSocket
- サーバーからの配信だけ → SSE
- リアルタイム不要 → ポーリング

スケールアウト時は、Redis Pub/Subなどでサーバー間の同期が必要。

「リアルタイムっぽい」ではなく、**本当にリアルタイム**を実現するために、WebSocketを正しく理解して使おう。
