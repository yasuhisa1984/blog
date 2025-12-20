---
title: "「コードには書いてない」を15分で暴く：Linux環境調査サバイバルガイド"
date: 2024-12-19
draft: false
categories: ["インフラ・運用"]
tags: ["Linux", "SSH", "cron", "運用", "トラブルシューティング", "チェックリスト"]
description: "cron、SSH、設定ファイル、環境変数...コードを見ても分からない「repo外運用」を、安全かつ最速で把握するためのLinux調査チェックリスト。現場で一生使える確認コマンド集。"
cover:
  image: "https://images.unsplash.com/photo-1629654297299-c8506221ca97?w=800"
  alt: "Linuxターミナル"
  relative: false
---

## 記事タイトル案（10個）

1. **「コードには書いてない」を15分で暴く：Linux環境調査サバイバルガイド** ← 採用
2. 「1年目で知るだろ」と言われる前に読む、Linux運用調査の全技術
3. repo外運用を丸裸にする：SSH・cron・設定ファイル完全調査マニュアル
4. マウントされないためのLinux調査術：初動15分で8割を潰す方法
5. 「見える化されてない」を技術で殴る：レガシー運用調査チェックリスト
6. コードだけ見ても分からない問題を、コマンド1発で解決する方法
7. cronが動かない、SSHが通らない、ログがない——全部この記事で解決する
8. アプリエンジニアのためのLinux運用調査入門：知らないと詰む15のポイント
9. 「どこに設定があるか分からない」を二度と言わないための調査チェックリスト
10. インフラ知識ゼロからの脱出：Linux環境を完全把握する実践ガイド

---

## TL;DR

- **コードを見ても分からない「repo外運用」** は、SSH設定・cron・環境変数・ログ出力先に埋まっている
- **最初の15分**で確認すべきことをチェックリスト化すれば、初動で8割は潰せる
- この記事の**コマンドはすべてread-only**（システムに影響を与えない）ので、本番でも安心して実行できる
- 「知らないのが悪い」ではなく「見える化されてないのが問題」——技術で解決する

---

## 目次

1. [この記事の前提と心構え](#この記事の前提と心構え)
2. [まず最初の15分でやること](#まず最初の15分でやること)
3. [カテゴリ別チェックリスト](#カテゴリ別チェックリスト)
   - ユーザー / グループ / ログイン履歴
   - SSH / 鍵 / 接続設定
   - Cron / スケジュールジョブ（見えないジョブも）
   - 設定ファイル / INI / 環境変数
   - ログ / 出力先
   - パーミッション / 所有者
   - ネットワーク / 疎通
4. [具体例：log-torukun監視スクリプトの調査](#具体例log-torukun監視スクリプトの調査)
5. [付録：深掘り調査テクニック](#付録深掘り調査テクニック)
6. [調査結果メモテンプレート](#調査結果メモテンプレート)
7. [「repo外運用」を減らすための改善ロードマップ](#repo外運用を減らすための改善ロードマップ)

---

## この記事の前提と心構え

### 想定シナリオ

```
あなたは PHP エンジニア。ある日、こう言われる。

上司「この監視スクリプト、たまに動かないんだけど調べて」
あなた「（コード見ても cron どこ...? SSH設定どこ...?）」
上司「え、1年目で知るだろそれくらい」
```

**コードだけ見ても分からない。** なぜなら：

- `~/.ssh/config` はリポジトリ外
- `crontab -l` の内容はリポジトリ外
- ログの出力先は cron のリダイレクト次第
- 実行ユーザーの環境変数は cron 環境では違う
- INI ファイルのパスはコードにハードコードされてるかも

### この記事のスタンス

```
❌「知らないのが悪い」
✅「見える化されてないのが問題」

→ だから、技術で殴って可視化する
```

### 安全に調査するための鉄則

```
⚠️ この記事のコマンドはすべて read-only

- cat, grep, ls, ps, lsof, ss, dig など「見るだけ」コマンド
- rm, mv, kill, systemctl restart などは使わない
- 本番環境でも安心して実行できる

万が一、変更が必要な場合は明示的に注記する
```

---

## まず最初の15分でやること

### 0. 自分が誰として作業しているか確認（30秒）

```bash
# 今のユーザーとホスト
whoami
hostname

# 現在のシェルと環境
echo $SHELL
echo $PATH
```

### 1. 問題のスクリプトの場所と中身を確認（2分）

```bash
# スクリプトの場所を確認
ls -la /path/to/script.php

# 中身を確認（先頭100行）
head -100 /path/to/script.php

# 設定ファイルの読み込み箇所を探す
grep -n "include\|require\|\.ini\|\.conf\|config" /path/to/script.php
```

### 2. そのスクリプトを動かしている cron を探す（3分）

```bash
# 現在ユーザーの crontab
crontab -l

# 他のユーザーの crontab（root権限必要）
sudo crontab -l -u log-torukun

# システム全体の cron
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/

# スクリプト名で grep
grep -r "script.php" /etc/cron* 2>/dev/null
sudo grep -r "script.php" /var/spool/cron/* 2>/dev/null
```

### 3. SSH 設定を確認（3分）

```bash
# 実行ユーザーの SSH 設定
sudo cat /home/log-torukun/.ssh/config
sudo ls -la /home/log-torukun/.ssh/

# known_hosts の確認
sudo cat /home/log-torukun/.ssh/known_hosts

# 鍵ファイルの確認
sudo ls -la /home/log-torukun/.ssh/*.pub
sudo ls -la /home/log-torukun/.ssh/id_*
```

### 4. 最近のログ・エラーを確認（3分）

```bash
# cron のログ（Ubuntu/Debian）
sudo tail -100 /var/log/syslog | grep -i cron

# cron のログ（RHEL/CentOS）
sudo tail -100 /var/log/cron

# スクリプトの出力先を cron から特定
crontab -l | grep "script.php"
# 例: * * * * * /usr/bin/php /path/to/script.php >> /var/log/myapp/cron.log 2>&1

# そのログを確認
sudo tail -100 /var/log/myapp/cron.log
```

### 5. プロセスの状態を確認（2分）

```bash
# 今動いているか
ps aux | grep "script.php"

# 関連プロセス
ps aux | grep "log-torukun"

# SSH 接続中のプロセス
ps aux | grep ssh
```

### 6. ネットワーク疎通を確認（2分）

```bash
# SSH 先への疎通
ping -c 3 <対象IP>

# ポート22への接続確認
nc -zv <対象IP> 22
# または
timeout 5 bash -c "</dev/tcp/<対象IP>/22" && echo "OK" || echo "NG"

# DNS解決
dig <対象ホスト名> +short
```

---

## カテゴリ別チェックリスト

### 1. ユーザー / グループ / ログイン履歴

#### 目的
サーバー上にどんなユーザーがいて、誰がどの権限で動いているかを把握する

#### 確認コマンド

```bash
# ========================================
# システムユーザーの一覧
# ========================================

# /etc/passwd からユーザー一覧を取得
cat /etc/passwd

# 人間が使うユーザーだけ抽出（UID 1000以上、一般的な基準）
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $6, $7}' /etc/passwd

# 特定ユーザーの詳細を確認
getent passwd log-torukun
# 出力例: log-torukun:x:1001:1001:Log Monitor User:/home/log-torukun:/bin/bash

# /etc/passwd の見方
# ユーザー名:パスワード:UID:GID:コメント:ホームディレクトリ:シェル
#
# シェルが /sbin/nologin や /bin/false → ログイン不可のサービスアカウント
# シェルが /bin/bash や /bin/sh → ログイン可能

# ========================================
# ログインできるユーザーを特定
# ========================================

# ログイン可能なシェルを持つユーザー
grep -v '/nologin\|/false\|/sync\|/shutdown\|/halt' /etc/passwd | cut -d: -f1

# sudo 権限を持つユーザー
grep -E '^sudo:|^wheel:' /etc/group
getent group sudo
getent group wheel  # RHEL/CentOS

# sudoers の設定確認
sudo cat /etc/sudoers
sudo ls -la /etc/sudoers.d/
sudo cat /etc/sudoers.d/*

# ========================================
# グループの確認
# ========================================

# グループ一覧
cat /etc/group

# 特定ユーザーの所属グループ
id log-torukun
groups log-torukun

# 特定グループのメンバー
getent group docker
getent group www-data

# ========================================
# ログイン履歴・現在の状態
# ========================================

# 現在ログイン中のユーザー
who
w

# 最近のログイン履歴
last | head -20

# 各ユーザーの最終ログイン日時
lastlog

# 失敗したログイン試行（セキュリティ確認）
sudo lastb | head -20

# ========================================
# サービスアカウントの特定
# ========================================

# cron を持っているユーザーを探す
for user in $(cut -d: -f1 /etc/passwd); do
  sudo crontab -l -u "$user" 2>/dev/null | grep -q . && echo "$user has crontab"
done

# ホームディレクトリがあるユーザー
ls -la /home/

# 特定ディレクトリの所有者から逆引き
stat -c "%U" /var/www/html/
stat -c "%U" /usr/local/tmp/log_check/
```

#### よくあるハマり

| 症状 | 原因 | 確認方法 |
|------|------|----------|
| 知らないユーザーでプロセスが動いている | サービスアカウント | `ps aux` + `getent passwd <user>` |
| sudo できない | sudo グループに入っていない | `id <user>`, `/etc/sudoers` |
| ログインできない | シェルが /sbin/nologin | `getent passwd <user>` |
| 「誰がこのファイル作った？」が分からない | 所有者から逆引き | `stat -c "%U" <file>` |

#### 次の一手

```bash
# 「このスクリプトを実行しているユーザー」を特定する流れ
1. ps aux | grep script.php          # 実行中ならユーザー名が見える
2. sudo crontab -l -u <user>         # そのユーザーの cron を確認
3. getent passwd <user>              # ユーザーの詳細（ホームディレクトリ等）
4. sudo ls -la /home/<user>/.ssh/    # SSH 設定があるか
```

---

### 2. SSH / 鍵 / 接続設定

#### 目的
SSHで外部サーバーに接続する処理が、どの設定・どの鍵で動いているかを把握する

#### 確認コマンド

```bash
# ========================================
# SSH設定ファイルの確認
# ========================================

# ユーザーの SSH config
cat ~/.ssh/config
sudo cat /home/<user>/.ssh/config

# SSH config で実際に解決される設定を見る（超重要）
ssh -G <host>

# 例：log-torukun ユーザーが worker-server に接続する設定
sudo -u log-torukun ssh -G worker-server

# ========================================
# 鍵ファイルの確認
# ========================================

# 鍵の一覧
ls -la ~/.ssh/

# 鍵のパーミッション（600でないとエラー）
stat -c "%a %U:%G %n" ~/.ssh/id_*

# 公開鍵の内容（接続先の authorized_keys と照合用）
cat ~/.ssh/id_rsa.pub
cat ~/.ssh/id_ed25519.pub

# ========================================
# known_hosts の確認
# ========================================

# 接続先が登録されているか
grep "<対象IP>" ~/.ssh/known_hosts
grep "<対象ホスト名>" ~/.ssh/known_hosts

# ========================================
# 接続テスト（デバッグモード）
# ========================================

# 詳細ログ付きで接続テスト
ssh -vvv <user>@<host> exit

# タイムアウト付き
ssh -o ConnectTimeout=5 <user>@<host> "echo OK"
```

#### よくあるハマり

| 症状 | 原因 | 確認方法 |
|------|------|----------|
| `Permission denied (publickey)` | 鍵が間違っている / 権限が緩い | `ssh -vvv` で使われている鍵を確認 |
| `Host key verification failed` | known_hosts に登録がない / 変わった | `~/.ssh/known_hosts` を確認 |
| `Connection timed out` | ネットワーク不通 / FW | `ping`, `nc -zv` で疎通確認 |
| `Could not resolve hostname` | DNS解決失敗 / config 未設定 | `ssh -G <host>` で HostName 確認 |
| cron では失敗するが手動では成功 | 鍵のパス / known_hosts が違う | `sudo -u <user> ssh ...` で再現 |

#### 次の一手

```bash
# 接続できない場合、段階的に切り分け
1. ping <ip>                    # L3疎通
2. nc -zv <ip> 22               # L4ポート
3. ssh -vvv user@ip exit        # SSH認証
4. sudo -u <cron-user> ssh ...  # ユーザー差異
```

---

### 3. Cron / スケジュールジョブ（見えないジョブも）

#### 目的
スクリプトがいつ・誰によって・どのように実行されているかを把握する

#### ⚠️ `crontab -l` で見えないケースがある

```bash
# crontab -l は「自分の crontab」しか見えない
crontab -l
# → 自分のジョブしか表示されない

# 以下は crontab -l では見えない：
# 1. 他のユーザーの crontab
# 2. /etc/cron.d/ 配下のファイル
# 3. /etc/cron.daily/ 等のディレクトリ
# 4. /etc/crontab（システム crontab）
# 5. anacron のジョブ
# 6. systemd timer
# 7. at コマンドで登録されたジョブ
```

#### 確認コマンド

```bash
# ========================================
# 1. ユーザー crontab（自分以外も）
# ========================================

# 自分の crontab
crontab -l

# 特定ユーザーの crontab（root権限必要）
sudo crontab -l -u log-torukun
sudo crontab -l -u www-data
sudo crontab -l -u root

# ★★★ 全ユーザーの crontab を一括確認 ★★★
for user in $(cut -d: -f1 /etc/passwd); do
  echo "=== $user ==="
  sudo crontab -l -u "$user" 2>/dev/null
done

# crontab ファイルの場所（直接確認）
# Ubuntu/Debian
sudo ls -la /var/spool/cron/crontabs/
sudo cat /var/spool/cron/crontabs/*

# RHEL/CentOS
sudo ls -la /var/spool/cron/
sudo cat /var/spool/cron/*

# ========================================
# 2. システム cron（/etc/cron*）
# ========================================

# /etc/crontab（システム全体の crontab）
cat /etc/crontab

# /etc/cron.d/ 配下（パッケージがインストールするジョブ）
ls -la /etc/cron.d/
cat /etc/cron.d/*

# 定期実行ディレクトリ（スクリプトを置くだけで動く）
ls -la /etc/cron.hourly/
ls -la /etc/cron.daily/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/

# 中身も確認
cat /etc/cron.daily/*
cat /etc/cron.hourly/*

# ========================================
# 3. anacron（電源オフ時も補完実行）
# ========================================

# anacron の設定
cat /etc/anacrontab

# anacron のタイムスタンプ（最終実行日時）
ls -la /var/spool/anacron/

# ========================================
# 4. at コマンド（一回限りのジョブ）
# ========================================

# 登録されている at ジョブ一覧
atq
sudo atq

# 特定ジョブの内容を確認
at -c <job-number>

# at ジョブの保存場所
sudo ls -la /var/spool/at/
sudo ls -la /var/spool/cron/atjobs/  # 一部ディストリ

# ========================================
# 5. systemd timer（cron の代替）
# ========================================

# 全タイマー一覧（次回実行時刻付き）
systemctl list-timers --all

# 特定タイマーの詳細
systemctl status <timer-name>.timer
systemctl cat <timer-name>.timer

# タイマーに紐づくサービスの確認
systemctl cat <timer-name>.service

# タイマーのログ
journalctl -u <timer-name>.timer
journalctl -u <timer-name>.service

# ========================================
# 6. 特定スクリプトを実行している cron を探す
# ========================================

# 全箇所を一括検索
grep -r "check_log" /etc/cron* 2>/dev/null
sudo grep -r "check_log" /var/spool/cron/* 2>/dev/null
systemctl list-timers --all | grep -i "check"

# 特定スクリプトがどこで呼ばれているか完全調査
script_name="check_log"
echo "=== /etc/crontab ===" && grep "$script_name" /etc/crontab
echo "=== /etc/cron.d/ ===" && grep -r "$script_name" /etc/cron.d/
echo "=== /etc/cron.*/ ===" && grep -r "$script_name" /etc/cron.hourly/ /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/ 2>/dev/null
echo "=== user crontabs ===" && sudo grep -r "$script_name" /var/spool/cron/ 2>/dev/null
echo "=== systemd timers ===" && systemctl list-timers --all | grep -i "$script_name"
```

#### よくあるハマり

| 症状 | 原因 | 確認方法 |
|------|------|----------|
| `crontab -l` で見えないのに動いている | /etc/cron.d/ や他ユーザーの crontab | 上記の全箇所検索を実行 |
| cron が実行されない | cron デーモン停止 | `systemctl status cron` |
| 手動では動くが cron では動かない | PATH が違う | cron 内で `PATH=` を明示 |
| 出力がどこにもない | リダイレクトされてない | cron 行の末尾を確認 |
| 「file not found」 | 相対パス問題 | 絶対パスに変更 |
| 権限エラー | 実行ユーザーの権限不足 | `sudo -u <user>` で再現 |
| 毎日実行のはずが動かない日がある | anacron がサーバー起動時に実行 | `/var/spool/anacron/` を確認 |
| 一度だけ動いて止まった | at コマンドで登録されていた | `atq` で確認 |

#### cron 環境の罠

```bash
# cron の環境変数を確認（非常に少ない）
# 以下を crontab に追加して確認
* * * * * env > /tmp/cron_env.txt

# 確認
cat /tmp/cron_env.txt
# → PATH=/usr/bin:/bin しかないことが多い

# 解決策：crontab 内で PATH を明示
PATH=/usr/local/bin:/usr/bin:/bin
* * * * * /path/to/script.sh
```

#### 次の一手

```bash
# cron で動かない時のデバッグ
1. crontab に `* * * * * /path/to/script.sh >> /tmp/cron_debug.log 2>&1` を追加
2. 1分待つ
3. /tmp/cron_debug.log を確認
4. エラーメッセージに応じて対処
```

---

### 4. 設定ファイル / INI / 環境変数

#### 目的
スクリプトが読み込む設定ファイルの場所・内容・権限を把握する

#### 確認コマンド

```bash
# ========================================
# 設定ファイルの場所を探す
# ========================================

# コード内で設定を読んでいる箇所を探す
grep -rn "\.ini\|\.conf\|\.env\|config" /path/to/app/

# 例：PHP の ini ファイル読み込み
grep -n "parse_ini" /path/to/script.php

# 環境変数参照
grep -n "getenv\|ENV\|\$_ENV\|\$_SERVER" /path/to/script.php

# ========================================
# 設定ファイルの確認
# ========================================

# 場所・所有者・パーミッション
ls -la /usr/local/tmp/log_check/*.ini

# 内容
cat /usr/local/tmp/log_check/targets.ini

# 機密情報がないか（パスワード等）
grep -i "pass\|secret\|key\|token" /path/to/config.ini

# ========================================
# 環境変数
# ========================================

# 現在の環境変数
env | sort

# 特定の環境変数
echo $DATABASE_URL
echo $API_KEY

# /etc/environment（システム全体）
cat /etc/environment

# /etc/profile.d/（ログイン時に読まれる）
ls -la /etc/profile.d/
```

#### よくあるハマり

| 症状 | 原因 | 確認方法 |
|------|------|----------|
| 設定ファイルが見つからない | パスが間違っている | コード内のパスを確認 |
| 設定が反映されない | 古いキャッシュ | 設定ファイルの mtime 確認 |
| 権限エラー | 実行ユーザーが読めない | `ls -la` + `sudo -u <user> cat` |
| 環境変数が空 | cron 環境では設定されない | cron 内で export |

#### 次の一手

```bash
# 設定ファイルの依存関係を追う
1. コードで読み込み箇所を特定
2. そのパスのファイルを確認
3. ファイル内で include/import している別ファイルを確認
4. 環境変数で上書きされていないか確認
```

---

### 5. ログ / 出力先

#### 目的
スクリプトの実行結果・エラーがどこに出力されているかを把握する

#### 確認コマンド

```bash
# ========================================
# cron の出力先を特定
# ========================================

# crontab の行を確認
crontab -l
# 例: * * * * * /path/to/script.sh >> /var/log/myapp.log 2>&1

# リダイレクト先を確認
tail -100 /var/log/myapp.log

# リダイレクトがない場合 → メールに送られる
# /var/mail/<user> を確認
cat /var/mail/log-torukun

# ========================================
# syslog / journal
# ========================================

# Ubuntu/Debian: syslog
sudo tail -200 /var/log/syslog | grep -i "cron\|<script-name>"

# RHEL/CentOS: messages
sudo tail -200 /var/log/messages | grep -i "cron\|<script-name>"

# systemd journal
journalctl -u cron --since "1 hour ago"
journalctl -t CRON --since "1 hour ago"

# ========================================
# アプリケーションログ
# ========================================

# よくある場所
ls -la /var/log/
ls -la /var/log/apache2/   # Debian系
ls -la /var/log/httpd/     # RHEL系
ls -la /var/log/nginx/
ls -la /var/log/php*/

# 最近更新されたログ
find /var/log -mmin -60 -type f 2>/dev/null

# ========================================
# プロセスが開いているファイル
# ========================================

# 特定プロセスのファイル
lsof -p <PID>

# 特定ユーザーが開いているファイル
lsof -u log-torukun

# 特定ファイルを開いているプロセス
lsof /var/log/myapp.log
```

#### よくあるハマり

| 症状 | 原因 | 確認方法 |
|------|------|----------|
| ログがない | リダイレクトされてない | cron 行を確認 |
| ログが古い | スクリプトが動いてない | `ls -la` で mtime 確認 |
| Permission denied | ログディレクトリの権限 | `ls -la /var/log/myapp/` |
| ディスクフル | 書き込めない | `df -h` |

#### 次の一手

```bash
# ログが見つからない場合
1. cron の行でリダイレクト先を確認
2. なければ /var/mail/<user> を確認
3. それでもなければ syslog/journal を確認
4. プロセスが動いているなら lsof -p <PID>
```

---

### 6. パーミッション / 所有者

#### 目的
実行ユーザーがファイル・ディレクトリにアクセスできるかを確認する

#### 確認コマンド

```bash
# ========================================
# ファイルの権限確認
# ========================================

# 基本
ls -la /path/to/file

# 数値で表示
stat -c "%a %U:%G %n" /path/to/file

# ディレクトリを再帰的に
ls -laR /path/to/dir/

# ========================================
# ユーザーが読める/実行できるか確認
# ========================================

# そのユーザーとして実行
sudo -u log-torukun cat /path/to/config.ini
sudo -u log-torukun ls -la /path/to/dir/

# ========================================
# 特殊なパーミッション
# ========================================

# SUID/SGID/Sticky bit を探す
find /path/to/app -perm /6000 -type f

# 実行権限のあるファイル
find /path/to/app -perm /111 -type f

# ========================================
# SSH 鍵の権限（重要）
# ========================================

# 正しい権限
# ~/.ssh/           → 700 (drwx------)
# ~/.ssh/config     → 600 (-rw-------)
# ~/.ssh/id_*       → 600 (-rw-------)
# ~/.ssh/*.pub      → 644 (-rw-r--r--)
# ~/.ssh/known_hosts → 644 (-rw-r--r--)
# ~/.ssh/authorized_keys → 600 (-rw-------)

stat -c "%a %n" ~/.ssh ~/.ssh/*
```

#### よくあるハマり

| 症状 | 原因 | 確認方法 |
|------|------|----------|
| Permission denied | 読み取り権限がない | `sudo -u <user> cat` |
| SSH鍵エラー | 鍵の権限が緩すぎる | `stat ~/.ssh/id_*` |
| 書き込みできない | ディレクトリに w がない | 親ディレクトリの権限 |
| 実行できない | x がない | `chmod +x` が必要 |

#### 次の一手

```bash
# 権限問題の切り分け
1. sudo -u <user> cat /path/to/file  # 読めるか
2. sudo -u <user> ls /path/to/dir/   # ディレクトリ見えるか
3. namei -l /path/to/file            # パス全体の権限を表示
```

---

### 7. ネットワーク / 疎通

#### 目的
SSH先やAPIエンドポイントへの接続が可能かを確認する

#### 確認コマンド

```bash
# ========================================
# L3 疎通（ICMP）
# ========================================

ping -c 3 <対象IP>
ping -c 3 <対象ホスト名>

# ========================================
# L4 疎通（TCP ポート）
# ========================================

# nc (netcat)
nc -zv <IP> 22
nc -zv <IP> 443

# bash 組み込み（nc がない場合）
timeout 5 bash -c "</dev/tcp/<IP>/22" && echo "OK" || echo "NG"

# telnet
telnet <IP> 22

# ========================================
# DNS 解決
# ========================================

# 基本
dig <hostname>
dig <hostname> +short

# 使用している DNS サーバー
cat /etc/resolv.conf

# 逆引き
dig -x <IP>

# ========================================
# ルーティング
# ========================================

# 経路確認
ip route
ip route get <対象IP>

# traceroute
traceroute <対象IP>
traceroute -n <対象IP>  # DNS解決しない

# ========================================
# ファイアウォール（ローカル）
# ========================================

# iptables
sudo iptables -L -n

# firewalld（RHEL/CentOS 7+）
sudo firewall-cmd --list-all

# ufw（Ubuntu）
sudo ufw status verbose

# ========================================
# 現在の接続状況
# ========================================

# 確立中の接続
ss -tnp
netstat -tnp  # 古い環境

# 特定ポートへの接続
ss -tnp | grep ":22"

# LISTEN しているポート
ss -tlnp
```

#### よくあるハマり

| 症状 | 原因 | 確認方法 |
|------|------|----------|
| ping は通るが SSH できない | FW で 22 がブロック | `nc -zv <ip> 22` |
| 名前解決できない | DNS 設定 | `dig`, `/etc/resolv.conf` |
| 接続がタイムアウト | 経路問題 / FW | `traceroute` |
| 接続が拒否される | サービス停止 / FW | 対象サーバー側を確認 |

#### 次の一手

```bash
# 接続できない場合の切り分け
1. ping <ip>           # ICMP通るか
2. nc -zv <ip> 22      # TCPポート通るか
3. dig <hostname>      # DNS解決できるか
4. traceroute <ip>     # どこで止まるか
5. 対象サーバーで ss -tlnp | grep 22  # Listen してるか
```

---

## 具体例：log-torukun監視スクリプトの調査

### シナリオ

```
監視スクリプト：
- html/cron/check_log_end.php
- /usr/local/tmp/log_check/*.ini を読む
- ssh <ip> -o 'ConnectTimeout=5' tail -n 1 <logName> で最終行取得
- 正常終了メッセージがなければ Slack 通知

SSHユーザー：log-torukun
問題：「たまに Slack 通知が来ない」
```

### 調査手順

#### Step 1：スクリプトの場所と中身を確認

```bash
# スクリプトの確認
cat /var/www/html/cron/check_log_end.php

# INI ファイルの読み込み箇所を探す
grep -n "\.ini" /var/www/html/cron/check_log_end.php
# → "/usr/local/tmp/log_check/*.ini" を読んでいることを確認
```

#### Step 2：INI ファイルの確認

```bash
# INI ファイルの一覧
ls -la /usr/local/tmp/log_check/

# 内容確認
cat /usr/local/tmp/log_check/targets.ini

# 例：
# [production-worker]
# host = 192.168.1.100
# log = /var/log/worker/batch.log
# success_pattern = "BATCH COMPLETED"

# パーミッション確認（log-torukun が読めるか）
sudo -u log-torukun cat /usr/local/tmp/log_check/targets.ini
```

#### Step 3：cron の確認

```bash
# log-torukun の crontab
sudo crontab -l -u log-torukun

# 例：
# */5 * * * * /usr/bin/php /var/www/html/cron/check_log_end.php >> /var/log/check_log/cron.log 2>&1

# ログの確認
sudo tail -100 /var/log/check_log/cron.log
```

#### Step 4：SSH 設定の確認

```bash
# log-torukun の SSH 設定
sudo cat /home/log-torukun/.ssh/config

# 例：
# Host production-worker
#   HostName 192.168.1.100
#   User log-torukun
#   IdentityFile ~/.ssh/id_ed25519
#   ConnectTimeout 5

# 鍵の確認
sudo ls -la /home/log-torukun/.ssh/

# known_hosts の確認
sudo cat /home/log-torukun/.ssh/known_hosts | grep "192.168.1.100"
```

#### Step 5：SSH 接続テスト

```bash
# log-torukun として接続テスト
sudo -u log-torukun ssh -o ConnectTimeout=5 production-worker "echo OK"

# 詳細ログ
sudo -u log-torukun ssh -vvv production-worker exit 2>&1 | head -50

# 実際のコマンドをテスト
sudo -u log-torukun ssh production-worker "tail -n 1 /var/log/worker/batch.log"
```

#### Step 6：ネットワーク疎通

```bash
# ping
ping -c 3 192.168.1.100

# SSH ポート
nc -zv 192.168.1.100 22
```

### 調査結果の例

```
■ 原因特定

1. /home/log-torukun/.ssh/known_hosts に 192.168.1.100 が登録されていなかった
2. 初回接続時に「Are you sure you want to continue connecting?」で止まっていた
3. cron はインタラクティブではないので、ここで失敗していた

■ 解決策

sudo -u log-torukun ssh-keyscan 192.168.1.100 >> /home/log-torukun/.ssh/known_hosts
```

---

## 付録：深掘り調査テクニック

### ssh -G で実際に解決される設定を見る

```bash
# ssh -G <host> で、config を解決した結果を表示
ssh -G production-worker

# 出力例：
# user log-torukun
# hostname 192.168.1.100
# port 22
# identityfile /home/log-torukun/.ssh/id_ed25519
# connecttimeout 5

# → コードに書かれた host名 が、実際にどの IP・ユーザー・鍵で接続されるか分かる
```

### ssh -vvv の読み方

```bash
ssh -vvv user@host exit 2>&1 | less

# 見るべきポイント：

# 1. 使用される鍵
debug1: Offering public key: /home/user/.ssh/id_ed25519 ED25519

# 2. 認証結果
debug1: Authentication succeeded (publickey).
# または
debug1: Authentications that can continue: publickey
debug1: No more authentication methods to try.  # ← 失敗

# 3. known_hosts
debug1: Host 'hostname' is known and matches the ED25519 host key.
# または
Host key verification failed.  # ← known_hosts にない

# 4. 接続先
debug1: Connecting to hostname [192.168.1.100] port 22.
```

### cron で動かない時の典型パターン

```bash
# ========================================
# パターン1：PATH が通っていない
# ========================================

# 症状
/usr/bin/php: not found

# 確認
crontab -l
# → php とだけ書いている

# 解決
# crontab の先頭に追加
PATH=/usr/local/bin:/usr/bin:/bin

# ========================================
# パターン2：相対パスで動かない
# ========================================

# 症状
require_once('./config.php'): failed to open stream

# 確認
# cron は / ディレクトリで実行される
pwd  # → /

# 解決
cd /var/www/html && /usr/bin/php script.php
# または
/usr/bin/php /var/www/html/script.php（絶対パス）

# ========================================
# パターン3：SSH 鍵が見つからない
# ========================================

# 症状
Permission denied (publickey).

# 確認
sudo -u log-torukun ssh -vvv host exit
# → 鍵のパスが違う / 権限が違う

# 解決
# ~/.ssh/config で IdentityFile を絶対パスで指定

# ========================================
# パターン4：known_hosts に登録がない
# ========================================

# 症状
Host key verification failed.

# 確認
grep "hostname" ~/.ssh/known_hosts

# 解決
ssh-keyscan hostname >> ~/.ssh/known_hosts

# ========================================
# パターン5：環境変数がない
# ========================================

# 症状
Error: DATABASE_URL is not set

# 確認
# cron は .bashrc を読まない

# 解決
# crontab 内で export
DATABASE_URL="postgres://..." /usr/bin/php script.php
# または
source /etc/environment && /usr/bin/php script.php
```

### 「repo外運用」を減らすための改善ロードマップ

```
Phase 0: 現状把握（この記事）
    └─ どこに何があるか調査・ドキュメント化

Phase 1: README 化
    └─ 設定ファイルの場所・cron の内容をリポジトリに記載
    └─ docs/operations.md を作成

Phase 2: リポジトリ管理化
    └─ crontab を cron.d ファイルとしてリポジトリ管理
    └─ SSH config をテンプレート化（機密情報は除く）
    └─ 設定ファイルをリポジトリに含める

Phase 3: 構成管理ツール導入
    └─ Ansible / Chef / Puppet で設定をコード化
    └─ サーバー構築の再現性を確保

Phase 4: 監視基盤への移行
    └─ Prometheus + Alertmanager
    └─ Datadog / Mackerel
    └─ cron + SSH 監視からの脱却
```

---

## 調査結果メモテンプレート

```markdown
# 調査結果メモ

## 基本情報
- 調査日時：YYYY-MM-DD HH:MM
- 調査者：
- 対象サーバー：
- 対象スクリプト：

## 1. スクリプト情報
- 場所：
- 実行ユーザー：
- 設定ファイル：
  - パス：
  - 権限：
  - 内容（要約）：

## 2. Cron 情報
- crontab 内容：
```
（ここに crontab -l の結果）
```
- ログ出力先：
- 最終実行時刻：

## 3. SSH 情報
- SSH config：
```
（ここに ssh -G host の結果）
```
- 鍵ファイル：
- known_hosts 登録：あり / なし
- 接続テスト結果：

## 4. ネットワーク
- 対象IP：
- ping：OK / NG
- port 22：OK / NG
- DNS解決：OK / NG

## 5. 発見した問題
1.
2.
3.

## 6. 解決策・対応内容
1.
2.
3.

## 7. 残課題・今後の改善
- [ ]
- [ ]
```

---

## まとめ

### この記事で伝えたかったこと

1. **「コードを見ても分からない」のは、あなたのせいではない**
   - repo外に運用が埋まっている構造が問題
   - 見える化されていないことを、技術で可視化する

2. **最初の15分で8割は潰せる**
   - whoami → crontab → SSH config → ログ → ネットワーク
   - この順番で確認すれば、大体の問題は特定できる

3. **すべて read-only で安全に確認できる**
   - 本番でも安心して実行できるコマンドだけを使う
   - 「調査したら壊れた」を防ぐ

4. **調査結果を残す習慣が、次の人を救う**
   - メモテンプレートを使って、調査結果をドキュメント化
   - 「repo外運用」を減らす第一歩

### 最後に

「1年目で知るだろ」と言われて悔しい思いをしたことがあるなら、この記事のチェックリストを使って、逆にマウントを取り返してほしい。

**知識は武器だ。見える化されていないものを見える化できるエンジニアは、現場で重宝される。**

この記事を保存して、次に「動かない」と言われた時に使ってほしい。

---

## 設計判断の背景

レガシー運用の調査は「知っているかどうか」で大きく効率が変わる。この記事では、経験則に頼らず「チェックリスト化」することで、誰でも同じ調査ができるようにした。特に「cron 環境の罠」「SSH の known_hosts 問題」は、初見で詰まる人が多いポイントなので重点的に解説した。

## 現場での判断基準

調査を依頼されたら、まず「そのスクリプトを誰が・いつ・どのように実行しているか」を確認する。これが分かれば、問題の8割は特定できる。コードだけ見て悩む時間を減らし、実行環境を見る時間を増やすことで、調査効率は劇的に上がる。

## 見るべきポイント

他のエンジニアの調査を見ていて「コードばかり見ている」場合は、「crontab -l と ~/.ssh/config は見た？」と聞くようにしている。この2つを見るだけで、大抵の「動かない」問題は解決への糸口が見つかる。
