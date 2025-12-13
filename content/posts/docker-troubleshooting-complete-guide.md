---
title: "実務で使うDocker障害調査の徹底ガイド"
date: 2025-12-13
draft: false
tags: ["Docker", "コンテナ", "障害対応", "トラブルシューティング", "運用", "インフラ", "実務"]
description: "コンテナが起動しない、OOMKilled、ネットワーク疎通不可、ディスク枯渇まで。本番Docker環境の障害調査を完全網羅"
cover:
  image: "images/covers/docker-containers.jpg"
  alt: "Docker障害調査"
  relative: false
---

## この記事の対象読者

- 「コンテナが起動しない」と言われて焦った経験がある人
- `docker logs` しか知らない人
- 本番環境でDockerを運用している/これからする人
- コンテナ内で何が起きているか分からず困った経験がある人

この記事では、**コンテナの状態確認**から、**ログ調査**、**リソース調査**、**ネットワーク調査**、**よくある障害パターン**まで、Docker障害調査を体系的に解説します。

---

## 障害調査の基本フロー

```
1. 状態確認    → コンテナは動いているか？
2. ログ確認    → 何が出力されているか？
3. リソース確認 → CPU/メモリ/ディスクは足りているか？
4. ネットワーク → 通信できているか？
5. 設定確認    → 環境変数、マウント、ポートは正しいか？
6. コンテナ内調査 → 中に入って確認
```

---

# 第1部：コンテナの状態確認

## docker ps：稼働中のコンテナ一覧

```bash
# 稼働中のコンテナ
docker ps

# 全てのコンテナ（停止中も含む）
docker ps -a

# 最近終了したコンテナ
docker ps -a --filter "status=exited"

# 特定の名前でフィルタ
docker ps -a --filter "name=web"

# フォーマットをカスタマイズ
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

出力例：
```
CONTAINER ID   IMAGE          COMMAND                  STATUS                      NAMES
abc123         nginx:latest   "/docker-entrypoint.…"   Up 2 hours                  web
def456         mysql:8        "docker-entrypoint.s…"   Exited (1) 5 minutes ago    db
ghi789         redis:7        "docker-entrypoint.s…"   Up 2 hours                  cache
```

## STATUSの読み方

| STATUS | 意味 | 対応 |
|--------|------|------|
| `Up X hours` | 正常稼働中 | - |
| `Exited (0)` | 正常終了 | 意図した終了か確認 |
| `Exited (1)` | エラー終了 | ログを確認 |
| `Exited (137)` | SIGKILL（OOMKilled等） | メモリ確認 |
| `Exited (143)` | SIGTERM（正常停止） | docker stopされた |
| `Restarting` | 再起動ループ | ログとヘルスチェック確認 |
| `Created` | 作成のみ（起動していない） | docker start が必要 |
| `Dead` | 削除に失敗 | docker rm -f で強制削除 |

## 終了コードの意味

```bash
# 終了コードを確認
docker inspect --format='{{.State.ExitCode}}' [container]
```

| 終了コード | 意味 |
|-----------|------|
| 0 | 正常終了 |
| 1 | アプリケーションエラー |
| 126 | コマンド実行権限なし |
| 127 | コマンドが見つからない |
| 128+n | シグナルnで終了 |
| 137 | SIGKILL (128+9) - OOMKilled |
| 143 | SIGTERM (128+15) - 正常停止 |
| 255 | 終了コード範囲外 |

## docker inspect：詳細情報

```bash
# 全情報を表示
docker inspect [container]

# 特定の情報だけ取得
docker inspect --format='{{.State.Status}}' [container]
docker inspect --format='{{.State.StartedAt}}' [container]
docker inspect --format='{{.RestartCount}}' [container]

# ネットワーク情報
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' [container]

# マウント情報
docker inspect --format='{{json .Mounts}}' [container] | jq

# 環境変数
docker inspect --format='{{json .Config.Env}}' [container] | jq

# ヘルスチェック状態
docker inspect --format='{{json .State.Health}}' [container] | jq
```

## OOMKilledの確認

```bash
# OOMKilledかどうか確認
docker inspect --format='{{.State.OOMKilled}}' [container]
# true ならメモリ不足で強制終了された

# メモリ制限を確認
docker inspect --format='{{.HostConfig.Memory}}' [container]
# 0 なら無制限
```

---

# 第2部：ログ調査

## docker logs：基本

```bash
# ログを表示
docker logs [container]

# 最新のN行
docker logs --tail 100 [container]

# リアルタイムで追従
docker logs -f [container]

# タイムスタンプ付き
docker logs -t [container]

# 特定の時間以降
docker logs --since "2025-12-13T10:00:00" [container]
docker logs --since "10m" [container]  # 10分前から

# 特定の時間まで
docker logs --until "2025-12-13T11:00:00" [container]

# 組み合わせ
docker logs -f --tail 100 -t [container]
```

## ログの検索

```bash
# エラーを検索
docker logs [container] 2>&1 | grep -i error

# 特定のリクエストを検索
docker logs [container] 2>&1 | grep "POST /api/users"

# 時間範囲でフィルタしてエラー検索
docker logs --since "1h" [container] 2>&1 | grep -i "exception"

# エラー前後の行も表示
docker logs [container] 2>&1 | grep -B 5 -A 10 "FATAL"
```

## ログドライバーの確認

```bash
# ログドライバーを確認
docker inspect --format='{{.HostConfig.LogConfig.Type}}' [container]
```

| ログドライバー | 説明 | docker logs |
|--------------|------|-------------|
| json-file | デフォルト、JSONファイル | ✅ 使える |
| local | 最適化されたローカルファイル | ✅ 使える |
| syslog | syslogに送信 | ❌ 使えない |
| journald | journaldに送信 | ❌ 使えない |
| fluentd | Fluentdに送信 | ❌ 使えない |
| awslogs | CloudWatch Logsに送信 | ❌ 使えない |

**注意:** syslog、fluentd等を使っている場合、`docker logs` は使えません。

## ログファイルの直接確認

```bash
# Dockerのログファイルの場所
/var/lib/docker/containers/[container-id]/[container-id]-json.log

# 巨大なログファイルの確認
ls -lh /var/lib/docker/containers/*/

# ログファイルを直接tail
tail -f /var/lib/docker/containers/[container-id]/*-json.log | jq

# ログファイルのサイズ（全コンテナ）
du -sh /var/lib/docker/containers/*/*.log | sort -h
```

## ログローテーション設定

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

```bash
# 設定を反映
systemctl restart docker
```

---

# 第3部：リソース調査

## docker stats：リアルタイム監視

```bash
# 全コンテナのリソース使用状況
docker stats

# 特定のコンテナ
docker stats [container1] [container2]

# 1回だけ表示（スクリプト用）
docker stats --no-stream

# フォーマットをカスタマイズ
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

出力例：
```
CONTAINER ID   NAME   CPU %    MEM USAGE / LIMIT     MEM %    NET I/O          BLOCK I/O
abc123         web    25.50%   512MiB / 1GiB         50.00%   1.2GB / 500MB    100MB / 50MB
def456         db     80.20%   2GiB / 2GiB           100.00%  500MB / 1.5GB    5GB / 2GB
```

## CPU使用率が高い場合

```bash
# コンテナ内のプロセスを確認
docker exec [container] ps aux --sort=-%cpu | head -20

# topで確認
docker exec -it [container] top

# htopがあれば
docker exec -it [container] htop
```

## メモリ使用量の詳細

```bash
# メモリ制限と使用量
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"

# cgroup から詳細を取得（コンテナ内から）
docker exec [container] cat /sys/fs/cgroup/memory/memory.usage_in_bytes
docker exec [container] cat /sys/fs/cgroup/memory/memory.limit_in_bytes

# OOM発生回数
docker exec [container] cat /sys/fs/cgroup/memory/memory.oom_control
```

## メモリ不足（OOMKilled）の調査

```bash
# システムログでOOMを確認
dmesg | grep -i "oom\|killed"

# journalctlで確認
journalctl -k | grep -i "oom\|killed"

# 特定のコンテナがOOMKilledされたか
docker inspect --format='{{.State.OOMKilled}}' [container]

# ホストのメモリ状況
free -h
```

## ディスク使用量

```bash
# Dockerが使用しているディスク容量
docker system df

# 詳細表示
docker system df -v
```

出力例：
```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          20        5         5.5GB     3.2GB (58%)
Containers      10        3         500MB     200MB (40%)
Local Volumes   15        8         10GB      2GB (20%)
Build Cache     50        0         2GB       2GB (100%)
```

```bash
# イメージのサイズ一覧
docker images --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}" | sort -k2 -h

# 使われていないリソースを削除
docker system prune        # 停止コンテナ、未使用ネットワーク、dangling イメージ
docker system prune -a     # 上記 + 使われていない全イメージ
docker volume prune        # 未使用ボリューム
docker builder prune       # ビルドキャッシュ
```

## コンテナ内のディスク使用量

```bash
# コンテナ内でディスク使用量確認
docker exec [container] df -h

# 大きいファイルを探す
docker exec [container] du -sh /* 2>/dev/null | sort -h

# 特定ディレクトリの詳細
docker exec [container] du -sh /var/log/*
```

---

# 第4部：ネットワーク調査

## ネットワークの基本確認

```bash
# Docker ネットワーク一覧
docker network ls

# ネットワークの詳細
docker network inspect [network-name]

# コンテナのIPアドレス
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' [container]

# 公開ポートの確認
docker port [container]
```

## コンテナ間の疎通確認

```bash
# コンテナAからコンテナBへping
docker exec [containerA] ping [containerB]
docker exec [containerA] ping [containerB-ip]

# コンテナ名で解決できるか
docker exec [containerA] nslookup [containerB]
docker exec [containerA] getent hosts [containerB]

# HTTPで疎通確認
docker exec [containerA] curl -v http://[containerB]:8080/health

# telnetでポート確認
docker exec [containerA] telnet [containerB] 3306
```

## ポートのリッスン状況

```bash
# コンテナ内でリッスンしているポート
docker exec [container] ss -tlnp
docker exec [container] netstat -tlnp

# ホストからコンテナのポートに接続
curl -v http://localhost:8080/
telnet localhost 8080
```

## DNS問題の調査

```bash
# コンテナのDNS設定
docker exec [container] cat /etc/resolv.conf

# DNS解決テスト
docker exec [container] nslookup google.com
docker exec [container] dig google.com

# Docker内蔵DNSサーバー（127.0.0.11）
docker exec [container] nslookup [other-container] 127.0.0.11
```

## ネットワークモードの確認

```bash
docker inspect --format='{{.HostConfig.NetworkMode}}' [container]
```

| モード | 説明 | 用途 |
|--------|------|------|
| bridge | デフォルト、仮想ブリッジ | 一般的なコンテナ |
| host | ホストのネットワークを共有 | パフォーマンス重視 |
| none | ネットワークなし | セキュリティ重視 |
| container:[name] | 他コンテナと共有 | サイドカー |

## tcpdumpでパケットキャプチャ

```bash
# コンテナ内でtcpdumpを実行（インストールが必要な場合あり）
docker exec [container] tcpdump -i eth0 -n port 80

# ホストからコンテナのネットワークをキャプチャ
# コンテナのvethインターフェースを特定
docker exec [container] cat /sys/class/net/eth0/iflink
# ホストで対応するvethを探す
ip link | grep [iflink番号]
# キャプチャ
tcpdump -i veth123abc -n port 80

# nsenterを使う方法
PID=$(docker inspect --format '{{.State.Pid}}' [container])
nsenter -t $PID -n tcpdump -i eth0 -n port 80
```

## iptablesの確認

```bash
# DockerのNATルール
iptables -t nat -L -n -v | grep DOCKER

# フィルタルール
iptables -L -n -v | grep DOCKER

# 特定ポートの転送ルール
iptables -t nat -L DOCKER -n -v
```

---

# 第5部：コンテナ内調査

## docker exec：コンテナに入る

```bash
# シェルで入る
docker exec -it [container] /bin/bash
docker exec -it [container] /bin/sh  # bashがない場合

# 特定ユーザーで実行
docker exec -u root -it [container] /bin/bash

# 環境変数を追加して実行
docker exec -e DEBUG=1 [container] /app/debug.sh

# ワーキングディレクトリを指定
docker exec -w /app [container] ls -la
```

## 停止したコンテナの調査

```bash
# 停止したコンテナからファイルをコピー
docker cp [container]:/var/log/app.log ./app.log
docker cp [container]:/etc/nginx/nginx.conf ./nginx.conf

# コンテナをイメージとして保存して調査
docker commit [container] debug-image
docker run -it debug-image /bin/bash

# コンテナのファイルシステムをtar出力
docker export [container] > container-fs.tar
tar tvf container-fs.tar | grep error
```

## デバッグ用コンテナを横に立てる

コンテナにデバッグツールがない場合：

```bash
# 同じネットワークにデバッグコンテナを起動
docker run -it --rm \
    --network container:[target-container] \
    nicolaka/netshoot \
    /bin/bash

# netshootには以下が入っている
# - curl, wget
# - dig, nslookup
# - tcpdump, tshark
# - iperf, netcat
# - strace, ltrace
# などなど
```

## プロセスの調査

```bash
# プロセス一覧
docker exec [container] ps aux

# プロセスツリー
docker exec [container] pstree -p

# 特定プロセスの詳細
docker exec [container] cat /proc/[pid]/status
docker exec [container] cat /proc/[pid]/cmdline

# ファイルディスクリプタ
docker exec [container] ls -la /proc/[pid]/fd/

# straceでシステムコール追跡
docker exec [container] strace -p [pid] -f
```

## ファイルシステムの調査

```bash
# 変更されたファイルを確認
docker diff [container]
# A = Added, C = Changed, D = Deleted

# マウントポイントの確認
docker exec [container] mount | grep -v "^proc\|^sys\|^dev"

# ファイルの権限確認
docker exec [container] ls -la /app/
docker exec [container] stat /app/config.yml

# 書き込み可能か確認
docker exec [container] touch /app/test && echo "writable" || echo "readonly"
```

---

# 第6部：Docker Compose環境

## docker compose での調査

```bash
# サービス一覧と状態
docker compose ps

# 全サービスのログ
docker compose logs

# 特定サービスのログ
docker compose logs [service]

# リアルタイム追従
docker compose logs -f [service]

# 直近のログ
docker compose logs --tail 100 [service]

# サービスの詳細
docker compose config  # 展開後の設定を表示
```

## サービス間の依存関係

```yaml
# docker-compose.yml
services:
  web:
    depends_on:
      db:
        condition: service_healthy  # ヘルスチェックを待つ
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  db:
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## ヘルスチェックの状態確認

```bash
# ヘルスチェック状態
docker inspect --format='{{json .State.Health}}' [container] | jq

# 出力例
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2025-12-13T10:00:00.000000000Z",
      "End": "2025-12-13T10:00:01.000000000Z",
      "ExitCode": 0,
      "Output": "OK"
    }
  ]
}
```

| Status | 意味 |
|--------|------|
| starting | ヘルスチェック中（start_period内） |
| healthy | 正常 |
| unhealthy | 異常 |

## 環境変数の確認

```bash
# docker-compose.yml の環境変数が正しく設定されているか
docker compose exec [service] env | sort

# 特定の環境変数
docker compose exec [service] echo $DATABASE_URL

# 秘匿情報を確認（注意して使用）
docker compose exec [service] printenv | grep -i password
```

## ボリュームの調査

```bash
# ボリューム一覧
docker volume ls

# ボリュームの詳細
docker volume inspect [volume-name]

# ボリュームの中身を確認
docker run --rm -v [volume-name]:/data alpine ls -la /data

# ボリュームのサイズ
docker run --rm -v [volume-name]:/data alpine du -sh /data
```

---

# 第7部：よくある障害パターン

## 1. コンテナが起動しない

### 症状

```bash
docker ps -a
# STATUS: Exited (1) ...
```

### 調査手順

```bash
# 1. ログを確認
docker logs [container]

# 2. 終了コードを確認
docker inspect --format='{{.State.ExitCode}}' [container]

# 3. コマンドを確認
docker inspect --format='{{.Config.Cmd}}' [container]
docker inspect --format='{{.Config.Entrypoint}}' [container]

# 4. 環境変数を確認
docker inspect --format='{{json .Config.Env}}' [container] | jq

# 5. シェルで起動してデバッグ
docker run -it --entrypoint /bin/sh [image]
```

### よくある原因

| 原因 | 解決策 |
|------|--------|
| コマンドが見つからない (127) | パス確認、イメージ確認 |
| 権限がない (126) | chmod +x、ユーザー確認 |
| 設定ファイルエラー | シンタックス確認 |
| 依存サービスに接続できない | depends_on、ヘルスチェック |
| ポートが既に使われている | 別のポートを使う |

## 2. コンテナがOOMKilledされる

### 症状

```bash
docker inspect --format='{{.State.OOMKilled}}' [container]
# true
```

### 調査手順

```bash
# 1. メモリ使用量を確認
docker stats --no-stream [container]

# 2. メモリ制限を確認
docker inspect --format='{{.HostConfig.Memory}}' [container]

# 3. システムログでOOM確認
dmesg | grep -i "oom\|killed" | tail -20

# 4. コンテナ内のメモリ消費プロセス
docker exec [container] ps aux --sort=-%mem | head -10
```

### 解決策

```yaml
# docker-compose.yml でメモリ制限を増やす
services:
  app:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G

# または docker run
docker run -m 2g --memory-reservation 1g [image]
```

## 3. コンテナ間で通信できない

### 症状

```bash
docker exec web curl http://api:8080
# curl: (6) Could not resolve host: api
```

### 調査手順

```bash
# 1. 両方のコンテナが同じネットワークにいるか
docker inspect --format='{{json .NetworkSettings.Networks}}' web | jq
docker inspect --format='{{json .NetworkSettings.Networks}}' api | jq

# 2. コンテナ名で解決できるか
docker exec web nslookup api

# 3. IPアドレスで接続できるか
docker exec web ping [api-ip-address]

# 4. ポートでリッスンしているか
docker exec api ss -tlnp
```

### 解決策

```yaml
# 同じネットワークに所属させる
services:
  web:
    networks:
      - app-network
  api:
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## 4. ディスクが足りない

### 症状

```bash
docker run [image]
# no space left on device
```

### 調査手順

```bash
# 1. ホストのディスク状況
df -h

# 2. Dockerが使用している容量
docker system df -v

# 3. 大きいログファイル
du -sh /var/lib/docker/containers/*/*.log | sort -h | tail -10

# 4. 大きいイメージ
docker images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" | sort -h | tail -10
```

### 解決策

```bash
# 未使用リソースを削除
docker system prune -a --volumes

# ログのサイズ制限（daemon.json）
{
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}

# 特定コンテナのログを削除（緊急時）
truncate -s 0 /var/lib/docker/containers/[container-id]/*-json.log
```

## 5. イメージのビルドが失敗する

### 症状

```bash
docker build -t myapp .
# ERROR: failed to solve: ...
```

### 調査手順

```bash
# 1. キャッシュなしでビルド
docker build --no-cache -t myapp .

# 2. 途中のステージで止める
docker build --target builder -t myapp:builder .

# 3. 失敗したレイヤーを調査
docker run -it [last-successful-layer] /bin/bash

# 4. BuildKitを無効にして詳細ログ
DOCKER_BUILDKIT=0 docker build -t myapp .
```

### よくある原因

| エラー | 原因 | 解決策 |
|--------|------|--------|
| COPY failed | ファイルがない/.dockerignore | パス確認 |
| RUN failed | コマンドエラー | シェルで確認 |
| network error | DNS/プロキシ | ネットワーク設定 |
| no space | ディスク不足 | docker system prune |

## 6. コンテナが遅い

### 症状

レスポンスが遅い、処理がタイムアウトする

### 調査手順

```bash
# 1. リソース使用状況
docker stats [container]

# 2. CPU制限を確認
docker inspect --format='{{.HostConfig.CpuQuota}}' [container]
docker inspect --format='{{.HostConfig.CpuPeriod}}' [container]
# CPU制限 = CpuQuota / CpuPeriod（0なら無制限）

# 3. I/O待ちを確認
docker exec [container] iostat -x 1 5

# 4. ネットワーク遅延を確認
docker exec [container] ping -c 10 [target]

# 5. ボリュームI/Oを確認
docker exec [container] dd if=/dev/zero of=/tmp/test bs=1M count=100 oflag=direct
```

### 解決策

```yaml
# リソース制限を緩和
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G

# または高速なストレージドライバを使用
# overlay2を使用しているか確認
docker info | grep "Storage Driver"
```

---

# 第8部：デバッグテクニック集

## 1. シグナルを送る

```bash
# SIGHUPを送る（設定再読み込み）
docker kill -s HUP [container]

# SIGUSRを送る（デバッグ情報出力）
docker kill -s USR1 [container]

# スレッドダンプを取得（Java）
docker exec [container] kill -3 1
docker logs [container] | grep -A 100 "Full thread dump"
```

## 2. コンテナをデバッグモードで再起動

```bash
# 環境変数を変えて再起動
docker run -it \
    --env DEBUG=true \
    --env LOG_LEVEL=debug \
    [image]

# エントリポイントを上書き
docker run -it --entrypoint /bin/bash [image]
```

## 3. Dockerイベントの監視

```bash
# リアルタイムでイベントを監視
docker events

# 特定のイベントをフィルタ
docker events --filter 'event=die'
docker events --filter 'container=[name]'
docker events --filter 'type=container'

# 時間範囲を指定
docker events --since '2025-12-13T10:00:00' --until '2025-12-13T11:00:00'
```

## 4. Dockerデーモンのログ

```bash
# systemdの場合
journalctl -u docker.service -f

# ログファイルの場合
tail -f /var/log/docker.log

# デバッグモードで起動
# /etc/docker/daemon.json
{
  "debug": true
}
systemctl restart docker
```

## 5. ホストからコンテナのプロセスを見る

```bash
# コンテナのPIDを取得
docker inspect --format='{{.State.Pid}}' [container]

# ホストからプロセスを確認
ps aux | grep [pid]

# /procを直接見る
ls -la /proc/[pid]/

# nsenterでコンテナの名前空間に入る
nsenter -t [pid] -n ip addr    # ネットワーク名前空間
nsenter -t [pid] -m ls /       # マウント名前空間
nsenter -t [pid] -p ps aux     # PID名前空間
```

## 6. レイヤーの調査

```bash
# イメージのレイヤーを確認
docker history [image]

# 各レイヤーのサイズ
docker history --no-trunc [image]

# 特定レイヤーの中身を確認
docker save [image] -o image.tar
tar xf image.tar
# 各レイヤーはblobs/sha256/以下にある
```

---

# 第9部：監視と予防

## docker healthcheck

```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

```yaml
# docker-compose.yml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 5s
```

## 再起動ポリシー

```yaml
services:
  app:
    restart: unless-stopped
    # no: 再起動しない
    # always: 常に再起動
    # on-failure: 異常終了時のみ
    # unless-stopped: 手動停止以外で再起動
```

## ログ集約

```yaml
# Fluentdにログを送信
services:
  app:
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.{{.Name}}
```

## Prometheus + cAdvisorで監視

```yaml
# docker-compose.yml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
```

---

## 障害調査チェックリスト

```
□ docker ps -a でコンテナの状態確認
□ docker logs でエラーログ確認
□ docker inspect で終了コード、OOMKilled確認
□ docker stats でリソース使用状況確認
□ docker network inspect でネットワーク確認
□ docker exec で中に入って調査
□ dmesg / journalctl でホストのログ確認
□ docker events でイベント履歴確認
```

---

## コマンド早見表

| 目的 | コマンド |
|------|---------|
| コンテナ状態 | `docker ps -a` |
| ログ確認 | `docker logs -f --tail 100 [c]` |
| 詳細情報 | `docker inspect [c]` |
| リソース | `docker stats [c]` |
| シェル | `docker exec -it [c] /bin/bash` |
| ファイルコピー | `docker cp [c]:/path ./` |
| ネットワーク | `docker network inspect [n]` |
| ディスク | `docker system df -v` |
| 掃除 | `docker system prune -a` |
| イベント | `docker events --filter container=[c]` |

---

## まとめ

### 調査の優先順位

```
1. ログを見る（docker logs）
2. 状態を見る（docker inspect）
3. リソースを見る（docker stats）
4. 中に入る（docker exec）
5. ホストを見る（dmesg, journalctl）
```

### 心がけ

1. **まずログを見る** — 9割の問題はログに答えがある
2. **終了コードを確認** — 137はOOM、143は正常停止
3. **リソース制限を意識** — メモリ/CPU制限を把握
4. **ネットワークは名前解決から** — 同じネットワークにいるか
5. **停止コンテナもデータは残る** — docker cp で救出可能

---

## 参考リンク

- [Docker公式: トラブルシューティング](https://docs.docker.com/config/daemon/)
- [Docker公式: ログ](https://docs.docker.com/config/containers/logging/)
- [nicolaka/netshoot](https://github.com/nicolaka/netshoot) - ネットワークデバッグコンテナ
- [cAdvisor](https://github.com/google/cadvisor) - コンテナ監視
- [Dive](https://github.com/wagoodman/dive) - イメージレイヤー分析
