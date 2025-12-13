---
title: "実務で使えるLinuxサーバー監視コマンド完全ガイド"
date: 2025-12-13
draft: false
tags: ["Linux", "サーバー監視", "インフラ", "運用", "コマンド", "トラブルシューティング"]
description: "CPU・メモリ・ディスク・ネットワーク・プロセス・コネクション監視の実践的なコマンドと読み方を網羅。現場で即使える知識を徹底解説"
cover:
  image: "images/covers/server-monitoring.jpg"
  alt: "Linuxサーバー監視"
  relative: false
---

## この記事について

この記事は、**実務でサーバーを監視・トラブルシューティングする**ためのLinuxコマンド完全ガイドです。

「サーバーが重い」「なんか遅い」と言われたときに、**何を見て、どう判断するか**を体系的にまとめました。

### 対象読者

- サーバー運用に携わるエンジニア
- 「top以外よく分からない」という人
- 障害対応で慌てたくない人
- インフラの基礎を固めたい人

### この記事で扱うこと

| カテゴリ | 内容 |
|----------|------|
| **CPU** | 負荷の見方、ボトルネックの特定 |
| **メモリ** | 使用量、スワップ、OOM |
| **ディスク** | 容量、I/O、ボトルネック |
| **ネットワーク** | コネクション、帯域、パケット |
| **プロセス** | 特定、追跡、強制終了 |
| **ログ** | 効率的な調査方法 |

---

## 監視の基本原則

### まず全体像を掴む

障害対応で最も重要なのは、**いきなり細部を見ない**こと。

```
1. 全体像を掴む（何がおかしいか？）
2. ボトルネックを特定する（どこがおかしいか？）
3. 原因を深掘りする（なぜおかしいか？）
4. 対処する（どう直すか？）
```

### リソースの4大要素

サーバーの問題は、ほぼこの4つに集約されます：

| リソース | 枯渇すると起きること |
|----------|----------------------|
| **CPU** | 処理が遅くなる、タイムアウト |
| **メモリ** | OOM Killer、スワップで激遅 |
| **ディスク** | 書き込み失敗、I/O待ち |
| **ネットワーク** | 接続失敗、タイムアウト |

---

# Part 1: CPU監視

## CPUの基本概念

### Load Average（負荷平均）とは

**Load Average** は、「CPUの待ち行列の長さ」を示します。

```bash
$ uptime
 14:30:01 up 30 days,  2:15,  3 users,  load average: 1.50, 2.00, 1.80
                                                      ^^^^  ^^^^  ^^^^
                                                      1分   5分   15分
```

#### Load Averageの読み方

| コア数 | 正常 | 要注意 | 危険 |
|--------|------|--------|------|
| 1コア | < 1.0 | 1.0〜2.0 | > 2.0 |
| 4コア | < 4.0 | 4.0〜8.0 | > 8.0 |
| 8コア | < 8.0 | 8.0〜16.0 | > 16.0 |

**目安**: Load Average ÷ CPUコア数 < 1.0 なら正常

```bash
# CPUコア数の確認
$ nproc
4

# または
$ grep -c processor /proc/cpuinfo
4
```

---

## uptime - 最速の状態確認

```bash
$ uptime
 14:30:01 up 30 days,  2:15,  3 users,  load average: 0.50, 0.80, 0.75
```

| 項目 | 意味 |
|------|------|
| `14:30:01` | 現在時刻 |
| `up 30 days` | 起動してからの時間 |
| `3 users` | ログイン中のユーザー数 |
| `load average` | 1分、5分、15分の負荷平均 |

### 負荷の傾向を読む

```
load average: 8.00, 2.00, 1.00
              ↑     ↑     ↑
              急上昇中！（直近で何かが起きた）

load average: 1.00, 2.00, 8.00
              ↑     ↑     ↑
              収束中（過去に何かあったが落ち着いた）

load average: 4.00, 4.00, 4.00
              ↑     ↑     ↑
              定常的に高い（慢性的な問題）
```

---

## top - リアルタイム監視の王道

```bash
$ top
```

### 画面の見方

```
top - 14:30:01 up 30 days,  2:15,  3 users,  load average: 0.50, 0.80, 0.75
Tasks: 200 total,   1 running, 199 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.0 us,  2.0 sy,  0.0 ni, 92.0 id,  1.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  16000.0 total,   8000.0 free,   6000.0 used,   2000.0 buff/cache
MiB Swap:   2000.0 total,   2000.0 free,      0.0 used.   9500.0 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1234 mysql     20   0  2.0g   500m   20m S  15.0   3.1   100:00.00 mysqld
 5678 www-data  20   0  300m   100m   50m S   5.0   0.6    50:00.00 php-fpm
```

### CPU行の詳細

```
%Cpu(s):  5.0 us,  2.0 sy,  0.0 ni, 92.0 id,  1.0 wa,  0.0 hi,  0.0 si,  0.0 st
          ^^^^     ^^^^     ^^^^    ^^^^^     ^^^^     ^^^^     ^^^^     ^^^^
```

| 項目 | 意味 | 高いとき |
|------|------|----------|
| **us** (user) | ユーザープロセス | アプリが重い |
| **sy** (system) | カーネル処理 | I/O多い、コンテキストスイッチ多い |
| **ni** (nice) | nice値変更プロセス | 優先度下げたプロセスが動いている |
| **id** (idle) | アイドル | 低いと忙しい |
| **wa** (wait) | I/O待ち | **ディスクがボトルネック** |
| **hi** (hardware interrupt) | ハードウェア割り込み | デバイスの問題 |
| **si** (software interrupt) | ソフトウェア割り込み | ネットワーク処理多い |
| **st** (steal) | VM盗まれた時間 | **ホストが過負荷**（クラウドで重要） |

### 重要な指標

**wa（I/O wait）が高い場合**
```
ディスクI/Oがボトルネック
→ iostatで詳細確認
→ 遅いクエリ、ログ出力過多などを疑う
```

**st（steal）が高い場合**
```
VMホストが過負荷（クラウド環境でよくある）
→ インスタンスタイプの変更を検討
→ クラウドプロバイダーに問い合わせ
```

### topの便利なキー操作

| キー | 動作 |
|------|------|
| `1` | CPU別表示（マルチコア環境で便利） |
| `M` | メモリ使用量でソート |
| `P` | CPU使用量でソート |
| `k` | プロセスをkill |
| `c` | コマンドライン全体表示 |
| `H` | スレッド表示 |
| `q` | 終了 |

---

## htop - より見やすいtop

```bash
$ htop
```

htopは、topの高機能版です。

### htopの利点

- カラー表示で見やすい
- マウス操作可能
- 横スクロールでコマンドライン全体が見える
- プロセスツリー表示（`F5`）
- 検索機能（`F3`）

### インストール

```bash
# Debian/Ubuntu
sudo apt install htop

# CentOS/RHEL
sudo yum install htop

# macOS
brew install htop
```

---

## vmstat - システム統計の定番

```bash
$ vmstat 1 5  # 1秒間隔で5回表示
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 800000  50000 200000    0    0     5    10  100  200  5  2 92  1  0
 2  0      0 795000  50000 200000    0    0     0    20  150  250 10  3 85  2  0
```

### 各項目の意味

#### procs（プロセス）

| 項目 | 意味 | 注意点 |
|------|------|--------|
| **r** | 実行待ちプロセス数 | CPUコア数より多いと過負荷 |
| **b** | I/O待ちプロセス数 | 0以外はディスクボトルネック |

#### memory（メモリ）

| 項目 | 意味 |
|------|------|
| swpd | 使用中のスワップ |
| free | 空きメモリ |
| buff | バッファキャッシュ |
| cache | ページキャッシュ |

#### swap（スワップ）

| 項目 | 意味 | 注意点 |
|------|------|--------|
| **si** | スワップイン | **0以外は危険信号** |
| **so** | スワップアウト | **0以外は危険信号** |

#### cpu

topと同じ。`wa`が高いとI/O待ち。

---

## mpstat - CPU別の詳細統計

```bash
$ mpstat -P ALL 1  # 全CPU、1秒間隔
```

```
CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal   %idle
all    5.00    0.00    2.00    1.00    0.00    0.00    0.00   92.00
  0   10.00    0.00    3.00    2.00    0.00    0.00    0.00   85.00
  1    2.00    0.00    1.00    0.00    0.00    0.00    0.00   97.00
  2    5.00    0.00    2.00    1.00    0.00    0.00    0.00   92.00
  3    3.00    0.00    2.00    1.00    0.00    0.00    0.00   94.00
```

### 使いどころ

- 特定のCPUだけ100%になっていないか確認
- シングルスレッドアプリのボトルネック発見

---

# Part 2: メモリ監視

## メモリの基本概念

### Linuxのメモリ管理

Linuxは、**空きメモリを積極的にキャッシュに使います**。

```
「空きメモリが少ない！」
  → 実はキャッシュとして有効活用されているだけかも
  → 本当に危険なのは「利用可能メモリ」が少ないとき
```

### メモリの種類

| 種類 | 説明 |
|------|------|
| **Used** | 実際に使用中 |
| **Free** | 完全に未使用 |
| **Buffers** | ブロックデバイスのキャッシュ |
| **Cached** | ファイルのキャッシュ |
| **Available** | 実際に利用可能な量（**これを見る**） |

---

## free - メモリ使用状況の確認

```bash
$ free -h  # 人間が読みやすい形式
```

```
              total        used        free      shared  buff/cache   available
Mem:           16Gi       6.0Gi       2.0Gi       500Mi       8.0Gi       9.0Gi
Swap:         2.0Gi          0B       2.0Gi
```

### 読み方のポイント

```
重要なのは「available」列！

available = free + (解放可能なbuff/cache)

この例では:
- total: 16GB
- available: 9GB（実際に使える量）
- → まだ余裕あり
```

### 危険な状態

```
              total        used        free      shared  buff/cache   available
Mem:           16Gi       15Gi       100Mi       500Mi       900Mi       500Mi
Swap:         2.0Gi       1.5Gi       500Mi
                          ^^^^                               ^^^^
                          スワップ使用中                      利用可能が少ない
```

**警告サイン:**
- `available` が総メモリの10%以下
- `Swap used` が0以外

---

## /proc/meminfo - 詳細なメモリ情報

```bash
$ cat /proc/meminfo
```

```
MemTotal:       16384000 kB
MemFree:         2000000 kB
MemAvailable:    9000000 kB
Buffers:          500000 kB
Cached:          7000000 kB
SwapCached:            0 kB
Active:          8000000 kB
Inactive:        5000000 kB
...
```

### よく見る項目

| 項目 | 意味 |
|------|------|
| MemTotal | 総物理メモリ |
| MemAvailable | 利用可能メモリ |
| SwapTotal | スワップ総量 |
| SwapFree | スワップ空き |
| Dirty | ディスク書き込み待ちのページ |
| Slab | カーネルのデータ構造用 |

---

## スワップの監視

### スワップが使われている＝危険信号

```bash
# スワップ使用状況
$ swapon --show
NAME      TYPE      SIZE   USED PRIO
/dev/sda2 partition   2G   500M   -2

# どのプロセスがスワップを使っているか
$ for pid in /proc/[0-9]*; do
    awk '/VmSwap/{if($2>0)print FILENAME,$0}' $pid/status 2>/dev/null
  done | sort -k3 -rn | head -10
```

### vmstatでスワップ活動を見る

```bash
$ vmstat 1
```

```
---swap--
  si   so
   0    0   ← 正常
 100  200   ← スワップイン/アウトが発生中（危険）
```

**si/soが継続的に0以外**: メモリ不足でスワップが常用されている

---

## OOM Killer

### OOM Killerとは

メモリが枯渇すると、Linuxカーネルは **OOM Killer** を発動し、プロセスを強制終了します。

### OOM Killerのログを確認

```bash
$ dmesg | grep -i "out of memory"
$ dmesg | grep -i "killed process"

# または
$ journalctl -k | grep -i oom
```

### OOM Killerの発動例

```
Out of memory: Killed process 1234 (java) total-vm:8000000kB, anon-rss:4000000kB
```

### 特定のプロセスをOOM Killerから守る

```bash
# OOMスコアを確認（高いほど殺されやすい）
$ cat /proc/1234/oom_score

# OOM Killerから保護（-1000 = 絶対に殺さない）
$ echo -1000 > /proc/1234/oom_score_adj
```

---

# Part 3: ディスク監視

## ディスクの基本概念

### 2つの観点

| 観点 | 内容 | 問題 |
|------|------|------|
| **容量** | どれだけ使っているか | 満杯になると書き込み失敗 |
| **I/O性能** | どれだけ速いか | 遅いとシステム全体が遅くなる |

---

## df - ディスク容量の確認

```bash
$ df -h  # 人間が読みやすい形式
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   35G   15G  70% /
/dev/sda2       100G   80G   20G  80% /var
tmpfs           7.8G  500M  7.3G   7% /dev/shm
```

### 重要なポイント

```bash
# 使用率が高いものだけ表示
$ df -h | awk '$5 > 80 {print}'
```

**警告ライン:**
- **80%**: 警告
- **90%**: 危険
- **95%以上**: 緊急対応

### inode使用量も確認

```bash
$ df -i
```

```
Filesystem      Inodes   IUsed   IFree IUse% Mounted on
/dev/sda1      3276800  500000 2776800   16% /
```

**小さいファイルが大量にある場合、inodeが先に枯渇することも**

---

## du - ディレクトリ別の使用量

```bash
# カレントディレクトリ直下の使用量
$ du -sh *

# 大きいディレクトリを探す
$ du -h --max-depth=1 / 2>/dev/null | sort -hr | head -20

# 特定ディレクトリ内で大きいファイルを探す
$ find /var/log -type f -size +100M -exec ls -lh {} \;
```

### よく容量を食う場所

```bash
/var/log           # ログファイル
/tmp               # 一時ファイル
/var/lib/docker    # Dockerイメージ・コンテナ
/home              # ユーザーデータ
```

---

## iostat - ディスクI/O統計

```bash
$ iostat -xz 1  # 拡張統計、1秒間隔
```

```
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %util
sda             10.00   50.00    100.00    500.00     0.00     5.00  15.00
sdb              5.00  100.00     50.00   2000.00     0.00    10.00  80.00
```

### 重要な指標

| 指標 | 意味 | 危険ライン |
|------|------|------------|
| **r/s** | 1秒あたりの読み取り回数 | - |
| **w/s** | 1秒あたりの書き込み回数 | - |
| **await** | I/O待ち時間（ms） | **> 20ms** |
| **%util** | ディスク使用率 | **> 80%** |

### await（I/O待ち時間）の目安

| デバイス | 正常 | 要注意 | 危険 |
|----------|------|--------|------|
| SSD | < 1ms | 1-5ms | > 5ms |
| HDD | < 10ms | 10-20ms | > 20ms |
| クラウドディスク | 変動大 | - | - |

### %utilが100%に張り付いている場合

```
ディスクが飽和状態
→ I/Oを減らすか、高速なディスクに変更
→ 遅いクエリ、過剰なログ出力を疑う
```

---

## iotop - プロセス別I/O使用量

```bash
$ sudo iotop -o  # I/Oを使っているプロセスのみ表示
```

```
Total DISK READ:       10.00 M/s | Total DISK WRITE:      50.00 M/s
  PID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 1234 be/4  mysql        5.00 M/s   30.00 M/s  0.00 %  80.00 % mysqld
 5678 be/4  root         1.00 M/s   10.00 M/s  0.00 %  15.00 % rsync
```

### どのプロセスがI/Oを使っているか一目瞭然

---

## lsof - 開いているファイルの確認

```bash
# 特定ディレクトリを開いているプロセス
$ lsof +D /var/log

# 削除されたけど開かれているファイル（容量を食い続ける）
$ lsof | grep deleted

# 特定プロセスが開いているファイル
$ lsof -p 1234
```

### 削除したのに容量が減らない問題

```bash
$ lsof | grep deleted
mysqld  1234  mysql  10u  REG  8,1  5000000000  /var/log/mysql.log (deleted)
                                    ^^^^^^^^^^
                                    5GB食っている

# プロセスを再起動するか、ファイルディスクリプタを閉じる
```

---

# Part 4: ネットワーク監視

## ネットワークの基本概念

### よくある問題

| 問題 | 症状 |
|------|------|
| **接続数過多** | 新規接続が拒否される |
| **帯域不足** | 転送が遅い |
| **パケットロス** | 通信が不安定 |
| **DNS問題** | 名前解決が遅い・失敗 |

---

## ss - 接続状態の確認（netstatの後継）

```bash
# 全接続を表示
$ ss -tuln

# 確立済み接続を表示
$ ss -t state established

# 特定ポートへの接続
$ ss -t dst :3306
```

### 接続状態の確認

```bash
$ ss -s
```

```
Total: 1500
TCP:   1200 (estab 800, closed 200, orphaned 50, timewait 150)
```

### 状態別の意味

| 状態 | 意味 | 多いと |
|------|------|--------|
| **ESTABLISHED** | 確立済み | 正常 |
| **TIME_WAIT** | 終了待ち | 短時間接続が多い |
| **CLOSE_WAIT** | 相手が閉じた | アプリがcloseしてない |
| **SYN_RECV** | SYN受信済み | SYN flood攻撃の可能性 |

### ポート別の接続数を確認

```bash
# ポート別の接続数
$ ss -t state established | awk '{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn | head
    500 3306    # MySQL
    300 80      # HTTP
    100 443     # HTTPS
```

### IPアドレス別の接続数

```bash
# 接続元IP別の接続数（DDoS調査など）
$ ss -t state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
    100 192.168.1.100
     50 192.168.1.101
     30 10.0.0.50
```

---

## netstat - 従来のネットワーク統計

```bash
# 接続状態の統計
$ netstat -s

# LISTEN中のポート
$ netstat -tlnp
```

### TCP統計の重要項目

```bash
$ netstat -s | grep -E "(retransmit|overflow|drop)"
    1234 segments retransmitted       # 再送が多いとネットワーク品質に問題
    56 times the listen queue of a socket overflowed  # 接続キューあふれ
```

---

## lsof - ネットワーク接続の詳細

```bash
# 特定ポートを使っているプロセス
$ lsof -i :3306

# 特定プロセスのネットワーク接続
$ lsof -i -a -p 1234

# 特定ホストへの接続
$ lsof -i @192.168.1.100
```

---

## tcpdump - パケットキャプチャ

```bash
# 特定ポートのトラフィック
$ sudo tcpdump -i eth0 port 80

# 特定ホストとの通信
$ sudo tcpdump -i eth0 host 192.168.1.100

# ファイルに保存（後でWiresharkで分析）
$ sudo tcpdump -i eth0 -w capture.pcap

# HTTPリクエストの内容を表示
$ sudo tcpdump -i eth0 -A port 80
```

### よく使うフィルタ

```bash
# SYNパケットのみ（接続開始）
$ sudo tcpdump 'tcp[tcpflags] & tcp-syn != 0'

# RSTパケット（接続拒否）
$ sudo tcpdump 'tcp[tcpflags] & tcp-rst != 0'

# 特定サイズ以上のパケット
$ sudo tcpdump 'greater 1000'
```

---

## ip / ifconfig - インターフェース情報

```bash
# インターフェース一覧と統計
$ ip -s link

# IPアドレス確認
$ ip addr

# ルーティングテーブル
$ ip route
```

### エラー統計の確認

```bash
$ ip -s link show eth0
```

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    RX: bytes  packets  errors  dropped  overrun  mcast
    1000000000 5000000  0       0        0        0
    TX: bytes  packets  errors  dropped  carrier  collsns
    500000000  2500000  0       0        0        0
                        ↑       ↑
                        エラーやドロップが増えていたら問題
```

---

## 帯域使用量の確認

### iftop - リアルタイム帯域モニタ

```bash
$ sudo iftop -i eth0
```

### nethogs - プロセス別帯域使用量

```bash
$ sudo nethogs eth0
```

```
  PID USER     PROGRAM                      DEV        SENT      RECEIVED
 1234 mysql    mysqld                       eth0       10.00 MB   5.00 MB
 5678 www-data nginx                        eth0        5.00 MB  20.00 MB
```

### nload - シンプルな帯域モニタ

```bash
$ nload eth0
```

---

# Part 5: プロセス監視

## ps - プロセス一覧

### 基本的な使い方

```bash
# 全プロセス（フル表示）
$ ps auxf

# 特定ユーザーのプロセス
$ ps -u mysql

# 特定コマンドのプロセス
$ ps aux | grep nginx

# 親子関係をツリー表示
$ ps auxf
```

### 出力の意味

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
mysql     1234  5.0  3.0 2000000 500000 ?      Ssl  Jan01 100:00 mysqld
          ^^^^  ^^^  ^^^  ^^^^^^ ^^^^^^        ^^^^
           │     │    │     │      │            └─ プロセス状態
           │     │    │     │      └─ 実メモリ使用量
           │     │    │     └─ 仮想メモリサイズ
           │     │    └─ メモリ使用率
           │     └─ CPU使用率
           └─ プロセスID
```

### STAT（プロセス状態）の意味

| 状態 | 意味 |
|------|------|
| **R** | 実行中 |
| **S** | スリープ（割り込み可能） |
| **D** | スリープ（割り込み不可、I/O待ち） |
| **Z** | ゾンビ |
| **T** | 停止 |

**D状態が多い** → I/Oボトルネック

---

## プロセスの特定テクニック

### CPU使用量トップ10

```bash
$ ps aux --sort=-%cpu | head -11
```

### メモリ使用量トップ10

```bash
$ ps aux --sort=-%mem | head -11
```

### 特定コマンドのプロセス数

```bash
$ pgrep -c nginx
8
```

### プロセスの起動時刻

```bash
$ ps -p 1234 -o pid,lstart,cmd
  PID                  STARTED CMD
 1234 Mon Jan  1 00:00:00 2024 /usr/sbin/mysqld
```

---

## pgrep / pkill - プロセスの検索と終了

```bash
# プロセス検索
$ pgrep -l nginx
1234 nginx
1235 nginx

# プロセスの詳細情報
$ pgrep -a nginx
1234 nginx: master process
1235 nginx: worker process

# プロセス終了（SIGTERM）
$ pkill nginx

# 強制終了（SIGKILL）
$ pkill -9 nginx
```

---

## プロセスの詳細調査

### /proc ファイルシステム

```bash
# プロセスのコマンドライン
$ cat /proc/1234/cmdline | tr '\0' ' '

# プロセスの環境変数
$ cat /proc/1234/environ | tr '\0' '\n'

# プロセスの開いているファイル
$ ls -la /proc/1234/fd

# プロセスのメモリマップ
$ cat /proc/1234/maps

# プロセスのステータス
$ cat /proc/1234/status
```

### strace - システムコールのトレース

```bash
# プロセスのシステムコールを追跡
$ strace -p 1234

# 新規プロセスを追跡
$ strace -f -e trace=network curl example.com

# ファイルアクセスのみ追跡
$ strace -e trace=file ls
```

---

# Part 6: ログ調査

## journalctl - systemdのログ

```bash
# 全ログ
$ journalctl

# 特定サービスのログ
$ journalctl -u nginx

# リアルタイム監視
$ journalctl -f

# 今日のログ
$ journalctl --since today

# エラーレベル以上
$ journalctl -p err

# カーネルログ
$ journalctl -k
```

### 時間指定

```bash
# 特定時間範囲
$ journalctl --since "2024-01-01 10:00:00" --until "2024-01-01 11:00:00"

# 直近1時間
$ journalctl --since "1 hour ago"
```

---

## ログファイルの効率的な調査

### tail - リアルタイム監視

```bash
# リアルタイム監視
$ tail -f /var/log/nginx/error.log

# 複数ファイル同時監視
$ tail -f /var/log/nginx/*.log

# 最後の100行
$ tail -n 100 /var/log/syslog
```

### grep - パターン検索

```bash
# エラーを検索
$ grep -i error /var/log/nginx/error.log

# 前後の行も表示
$ grep -B 5 -A 5 "error" /var/log/nginx/error.log

# 複数パターン
$ grep -E "(error|warning|critical)" /var/log/syslog

# 特定パターンを除外
$ grep -v "健全性チェック" /var/log/nginx/access.log
```

### awk - ログの集計

```bash
# アクセスログからステータスコード集計
$ awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
  10000 200
   1000 304
    500 404
    100 500

# IPアドレス別アクセス数
$ awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

### less - 大きいログの閲覧

```bash
$ less /var/log/syslog

# 検索: /パターン
# 次の検索結果: n
# 前の検索結果: N
# 末尾に移動: G
# 先頭に移動: g
```

---

## dmesg - カーネルログ

```bash
# カーネルログ表示
$ dmesg

# 人間が読みやすい時刻表示
$ dmesg -T

# エラーレベル以上
$ dmesg --level=err,warn

# リアルタイム監視
$ dmesg -w
```

### よく見るカーネルメッセージ

```bash
# OOM Killer
$ dmesg | grep -i "out of memory"

# ディスクエラー
$ dmesg | grep -i "error"

# ネットワークエラー
$ dmesg | grep -i "eth0"
```

---

# Part 7: 実践的なトラブルシューティング

## シナリオ1: 「サーバーが重い」

### 調査手順

```bash
# 1. まず全体像を把握
$ uptime
load average: 15.00, 10.00, 5.00  # 急上昇中

# 2. 何が原因か切り分け
$ top
# → %CPU, %wa, %st を確認

# 3a. CPU使用率が高い場合
$ ps aux --sort=-%cpu | head -5  # CPUを使っているプロセス特定

# 3b. I/O wait（wa）が高い場合
$ iostat -xz 1  # ディスクI/Oを確認
$ iotop -o      # どのプロセスがI/Oを使っているか

# 3c. steal（st）が高い場合
# → VMホストが過負荷、クラウドならインスタンス変更を検討
```

---

## シナリオ2: 「メモリ不足」

### 調査手順

```bash
# 1. メモリ状況確認
$ free -h
# → availableを見る

# 2. スワップが使われているか
$ vmstat 1
# → si/soが0以外なら危険

# 3. メモリを使っているプロセス特定
$ ps aux --sort=-%mem | head -10

# 4. OOM Killerが発動していないか
$ dmesg | grep -i "out of memory"

# 5. プロセス別の詳細メモリ使用量
$ cat /proc/1234/status | grep -E "(VmRSS|VmSwap)"
```

---

## シナリオ3: 「ディスクが満杯」

### 調査手順

```bash
# 1. どのパーティションが満杯か
$ df -h

# 2. 大きいディレクトリを探す
$ du -h --max-depth=1 / 2>/dev/null | sort -hr | head -20

# 3. 大きいファイルを探す
$ find / -type f -size +1G 2>/dev/null

# 4. 削除されたけど開かれているファイル
$ lsof | grep deleted

# 5. ログローテーションを確認
$ ls -lh /var/log/
```

### よくある原因と対処

| 原因 | 対処 |
|------|------|
| ログファイル肥大 | logrotate設定、古いログ削除 |
| Dockerイメージ | `docker system prune` |
| 一時ファイル | `/tmp`のクリーンアップ |
| コアダンプ | `/var/crash`の確認 |

---

## シナリオ4: 「接続できない」

### 調査手順

```bash
# 1. サービスが起動しているか
$ systemctl status nginx

# 2. ポートがLISTENしているか
$ ss -tlnp | grep :80

# 3. ファイアウォール
$ iptables -L -n
$ ufw status

# 4. 接続数が上限に達していないか
$ ss -s

# 5. ログにエラーがないか
$ journalctl -u nginx --since "10 minutes ago"
```

---

## シナリオ5: 「特定のプロセスが暴走」

### 調査手順

```bash
# 1. プロセスの特定
$ top  # Pキーで CPU順ソート

# 2. プロセスの詳細
$ ps -p 1234 -o pid,ppid,user,%cpu,%mem,start,cmd

# 3. 何をしているか
$ strace -p 1234

# 4. 開いているファイル
$ lsof -p 1234

# 5. 必要なら停止
$ kill 1234      # 正常終了
$ kill -9 1234   # 強制終了
```

---

# Part 8: 監視のベストプラクティス

## 定常監視で見るべき項目

| カテゴリ | 指標 | 閾値（目安） |
|----------|------|--------------|
| CPU | Load Average / コア数 | < 1.0 |
| CPU | I/O wait | < 10% |
| メモリ | Available | > 20% |
| メモリ | Swap使用量 | = 0 |
| ディスク | 使用率 | < 80% |
| ディスク | I/O await | < 20ms |
| ネットワーク | 接続数 | 上限の80%以下 |
| プロセス | ゾンビプロセス | = 0 |

---

## ワンライナー集

### 全体状況の確認

```bash
# 1コマンドで全体像
$ echo "=== Load ===" && uptime && echo "=== Memory ===" && free -h && echo "=== Disk ===" && df -h && echo "=== Top Processes ===" && ps aux --sort=-%cpu | head -5
```

### 異常検知

```bash
# Load Averageが高いかチェック
$ [ $(awk '{print int($1)}' /proc/loadavg) -gt $(nproc) ] && echo "HIGH LOAD"

# ディスク使用率80%超えチェック
$ df -h | awk '$5 > 80 {print "WARNING:", $0}'

# スワップが使われているかチェック
$ [ $(free | awk '/Swap/{print $3}') -gt 0 ] && echo "SWAP IN USE"
```

---

## まとめ

この記事で紹介したコマンドを、目的別にまとめます：

### まず実行するコマンド

| 目的 | コマンド |
|------|----------|
| 全体状況 | `uptime`, `top`, `htop` |
| メモリ | `free -h` |
| ディスク容量 | `df -h` |
| ディスクI/O | `iostat -xz 1` |
| ネットワーク | `ss -s` |
| ログ | `journalctl -p err --since "1 hour ago"` |

### 深掘りするコマンド

| 目的 | コマンド |
|------|----------|
| CPU詳細 | `mpstat -P ALL 1`, `vmstat 1` |
| メモリ詳細 | `/proc/meminfo`, `vmstat 1` |
| ディスク詳細 | `iotop`, `lsof` |
| ネットワーク詳細 | `ss -t`, `tcpdump`, `nethogs` |
| プロセス詳細 | `ps auxf`, `strace -p PID`, `lsof -p PID` |

---

## 参考資料

- [Linux Performance](http://www.brendangregg.com/linuxperf.html) - Brendan Gregg
- [USE Method](http://www.brendangregg.com/usemethod.html) - リソース分析手法
- [Linux Performance Analysis in 60 Seconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55) - Netflix Tech Blog

---

この記事が、サーバー運用の現場で役立てば幸いです。

「何を見ればいいか分からない」という状態から、「ここを見れば分かる」という状態になれることを目指しました。
