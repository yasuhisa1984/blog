---
title: "dotfilesとは何か？僕の開発環境と仕事の流儀"
date: 2025-12-13
draft: false
categories: ["AI・開発ツール"]
tags: ["dotfiles", "vim", "Neovim", "tmux", "wezterm", "開発環境", "初心者向け"]
description: "dotfilesの基本から、Neovim・tmux・WeTermの使い方、なぜこの構成を選んだのかまで。開発環境に現れる仕事の流儀を解説"
cover:
  image: "images/covers/terminal.jpg"
  alt: "dotfilesと仕事の流儀"
  relative: false
---

## この記事の対象読者

- 「dotfiles」という言葉を聞いたことがあるけど、よく分からない人
- 開発環境を整えたいけど、何から始めればいいか分からない人
- Vim/Neovim、tmuxに興味があるけど、敷居が高いと感じている人

この記事では、**dotfilesの基本概念**から、**僕が実際に使っているツール**、そして**なぜこの構成を選んだのか**という仕事の流儀まで解説します。

---

## dotfilesとは何か？

### 一言で言うと

**dotfiles（ドットファイル）** とは、Unix/Linux/macOSで使われる **設定ファイル** のことです。

ファイル名が `.`（ドット）で始まるため、dotfilesと呼ばれます。

### 具体例

```
~/.bashrc      → シェル（Bash）の設定
~/.zshrc       → シェル（Zsh）の設定
~/.vimrc       → Vimエディタの設定
~/.gitconfig   → Gitの設定
~/.tmux.conf   → tmuxの設定
```

### なぜ「隠しファイル」なのか？

Unix系OSでは、`.` で始まるファイルは **デフォルトで非表示** になります。

```bash
ls        # 通常のファイルのみ表示
ls -a     # dotfiles（隠しファイル）も表示
```

設定ファイルは普段触らないので、邪魔にならないよう隠されています。

---

## なぜdotfilesを管理するのか？

### メリット

| メリット | 説明 |
|----------|------|
| **環境の再現** | 新しいPCでも同じ環境をすぐ構築できる |
| **バックアップ** | 設定を失う心配がない |
| **バージョン管理** | Gitで変更履歴を追える |
| **共有** | チームで設定を共有できる |

### 一般的な管理方法

```bash
# 1. dotfilesをGitリポジトリで管理
~/dotfiles/
├── .vimrc
├── .tmux.conf
├── .zshrc
└── install.sh   # シンボリックリンクを張るスクリプト

# 2. ホームディレクトリにシンボリックリンクを作成
ln -s ~/dotfiles/.vimrc ~/.vimrc
```

#### dotfiles管理のワークフロー

```mermaid
flowchart LR
    Create["📝 設定ファイル作成<br/>━━━━━━<br/>.vimrc<br/>.tmux.conf<br/>.zshrc"] --> Repo["📁 Gitリポジトリ化<br/>━━━━━━<br/><code style='color: white'>~/dotfiles/</code><br/><br/><code style='color: white'>git init</code><br/><code style='color: white'>git add .</code><br/><code style='color: white'>git commit</code>"]

    Repo --> Remote["☁️ GitHub/GitLab<br/>にプッシュ<br/>━━━━━━<br/>バックアップ<br/>どこからでもアクセス"]

    Repo --> Link["🔗 シンボリックリンク<br/>━━━━━━<br/><code style='color: white'>ln -s ~/dotfiles/.vimrc</code><br/><code style='color: white'>      ~/.vimrc</code><br/><br/>ホームディレクトリから<br/>dotfilesを参照"]

    Link --> NewPC["💻 新しいPC<br/>━━━━━━<br/>1. <code style='color: white'>git clone</code><br/>2. <code style='color: white'>./install.sh</code><br/><br/>3秒で環境再現！"]

    Remote -.->|clone| NewPC

    Update["🔄 設定を更新"] --> Repo
    Repo -.-> Remote

    style Create fill:#e3f2fd
    style Repo fill:#fff3e0
    style Remote fill:#e8f5e9
    style Link fill:#fff3e0
    style NewPC fill:#e8f5e9
    style Update fill:#e3f2fd
```

---

## 僕の開発環境の全体像

```mermaid
flowchart TB
    subgraph WezTerm["🖥️ WezTerm（ターミナルエミュレータ）"]
        WT["作業場の「建物」<br/>━━━━━━<br/>・GPU加速<br/>・Lua設定<br/>・クロスプラットフォーム"]
    end

    subgraph Tmux["📦 tmux（ターミナル多重化）"]
        TM["作業机の「配置」<br/>━━━━━━<br/>・セッション管理<br/>・画面分割<br/>・永続化"]
    end

    subgraph Apps["🛠️ アプリケーション"]
        direction LR
        NV["📝 Neovim<br/>━━━━━━<br/>メインの「道具」<br/><br/>・テキスト編集<br/>・LSP<br/>・プラグイン"]
        ZSH["⚡ zsh<br/>━━━━━━<br/>コマンドを打つ「場所」<br/><br/>・ログ監視<br/>・コマンド実行<br/>・Git操作"]
    end

    WezTerm --> Tmux
    Tmux --> Apps

    Note["💡 レイヤー構造<br/>━━━━━━<br/>外側ほど「入れ物」<br/>内側ほど「作業内容」<br/><br/>壊れても各層で復旧可能"]

    style WezTerm fill:#e3f2fd
    style Tmux fill:#fff3e0
    style Apps fill:#e8f5e9
    style NV fill:#c8e6c9
    style ZSH fill:#c8e6c9
    style Note fill:#fffde7
```

| ツール | 役割 | 一言で |
|--------|------|--------|
| **WezTerm** | ターミナルエミュレータ | 「作業場の建物」 |
| **tmux** | ターミナル多重化 | 「作業机の配置」 |
| **Neovim** | テキストエディタ | 「メインの道具」 |
| **zsh** | シェル | 「コマンドを打つ場所」 |

---

## 各ツールの解説

### 1. WezTerm（ターミナルエミュレータ）

#### ターミナルとは？

**ターミナル（端末）** は、コマンドを入力してコンピュータを操作するためのアプリです。

Windowsの「コマンドプロンプト」、macOSの「ターミナル.app」のようなものです。

#### WezTermを選んだ理由

| 特徴 | 説明 |
|------|------|
| **軽量** | 起動が速く、メモリ消費が少ない |
| **安定** | 落ちない、固まらない |
| **Lua設定** | 設定ファイルがLuaで書ける（プログラマブル） |
| **クロスプラットフォーム** | Windows/Mac/Linux対応 |

#### 設定例（~/.wezterm.lua）

```lua
local wezterm = require 'wezterm'
local config = {}

config.font = wezterm.font 'JetBrains Mono'
config.font_size = 14.0
config.color_scheme = 'Tokyo Night'

-- 透過設定
config.window_background_opacity = 0.95

return config
```

**公式サイト:** https://wezfurlong.org/wezterm/

---

### 2. tmux（ターミナル多重化ツール）

#### tmuxとは？

**tmux** は、1つのターミナル内で **複数の画面を管理** できるツールです。

#### 何がうれしいのか？

```
通常のターミナル:
┌──────────────┐
│              │  ← 1画面だけ
│   シェル     │
│              │
└──────────────┘

tmuxを使うと:
┌──────────┬──────────┐
│          │          │
│  エディタ │   ログ   │  ← 分割できる
│          │          │
├──────────┴──────────┤
│     コマンド実行     │
└─────────────────────┘
```

#### tmuxの主要概念

| 概念 | 説明 |
|------|------|
| **セッション** | 作業全体のまとまり（プロジェクト単位） |
| **ウィンドウ** | タブのようなもの |
| **ペイン** | 画面の分割 |

##### セッション・ウィンドウ・ペインの階層構造

```mermaid
flowchart TB
    subgraph Session["📁 セッション（work）"]
        Note1["プロジェクト全体のまとまり<br/>SSHが切れても生き続ける"]

        subgraph Window1["📑 ウィンドウ1（editor）"]
            W1["タブのようなもの<br/><code style='color: white'>Ctrl+b c</code> で作成"]

            subgraph Panes1["画面分割"]
                direction LR
                P1["📝 ペイン1<br/>━━━━━━<br/>Neovim<br/>エディタ"]
                P2["⚡ ペイン2<br/>━━━━━━<br/>シェル<br/>コマンド実行"]
            end
        end

        subgraph Window2["📑 ウィンドウ2（server）"]
            W2["<code style='color: white'>Ctrl+b n</code> で切り替え"]

            subgraph Panes2["画面分割"]
                direction LR
                P3["🖥️ ペイン1<br/>━━━━━━<br/>サーバー起動<br/><code style='color: white'>npm run dev</code>"]
                P4["📊 ペイン2<br/>━━━━━━<br/>ログ監視<br/><code style='color: white'>tail -f app.log</code>"]
            end
        end
    end

    Detach["🔌 セッションから離脱<br/><code style='color: white'>Ctrl+b d</code><br/><br/>→ 作業は続行中"]
    Attach["🔗 セッションに再接続<br/><code style='color: white'>tmux attach -t work</code><br/><br/>→ 作業の続きから"]

    Session -.-> Detach
    Detach -.-> Attach
    Attach -.-> Session

    style Session fill:#e3f2fd
    style Window1 fill:#fff3e0
    style Window2 fill:#fff3e0
    style Panes1 fill:#e8f5e9
    style Panes2 fill:#e8f5e9
    style Detach fill:#ffebee
    style Attach fill:#c8e6c9
```

#### 最大のメリット：セッションの永続化

```bash
# SSHで接続中に回線が切れても...
# tmuxのセッションは生き続ける

tmux attach  # 再接続すれば、作業の続きからできる
```

これが「**安心に近い感覚**」の正体です。

#### 基本的な使い方

```bash
tmux                    # 新しいセッションを開始
tmux new -s work        # "work"という名前でセッション開始
tmux ls                 # セッション一覧
tmux attach -t work     # "work"セッションに接続
```

#### キーバインド（デフォルト）

| キー | 動作 |
|------|------|
| `Ctrl+b %` | 縦分割 |
| `Ctrl+b "` | 横分割 |
| `Ctrl+b o` | ペイン移動 |
| `Ctrl+b d` | セッションから離脱 |
| `Ctrl+b c` | 新しいウィンドウ |

**公式サイト:** https://github.com/tmux/tmux

---

### 3. Neovim（テキストエディタ）

#### VimとNeovimの違い

| | Vim | Neovim |
|--|-----|--------|
| 歴史 | 1991年〜 | 2014年〜（Vimからフォーク） |
| 設定言語 | Vimscript | Vimscript + **Lua** |
| プラグイン | 豊富 | 豊富 + モダンなエコシステム |
| 非同期処理 | 後から追加 | 最初から対応 |

**Neovim**は、Vimをモダンに書き直したプロジェクトです。

#### なぜVim系エディタを使うのか？

VS Codeではなく、あえてVim系を使う理由：

| 理由 | 説明 |
|------|------|
| **キーボード完結** | マウスに手を伸ばす必要がない |
| **起動が一瞬** | 0.1秒で立ち上がる |
| **どこでも使える** | SSH先のサーバーでも同じ操作感 |
| **思考が止まらない** | 「操作」ではなく「編集」に集中できる |

#### 起動速度の比較

```
Neovim:  ~0.05秒
VS Code: ~3-5秒（プロジェクトによる）
```

これは設計思想の違いです：

- **Neovim**: 必要最小限で起動 → 後から機能を読み込む
- **VS Code**: 全部準備してから起動

#### モード（Vimの基本概念）

Vimには「モード」という概念があります：

| モード | 用途 | 切り替え |
|--------|------|----------|
| **ノーマル** | 移動・編集コマンド | `Esc` |
| **インサート** | 文字入力 | `i`, `a`, `o` |
| **ビジュアル** | 選択 | `v`, `V` |
| **コマンド** | 保存・終了など | `:` |

##### Vimのモード遷移

```mermaid
flowchart TB
    Normal["⚙️ ノーマルモード<br/>━━━━━━<br/>移動・編集コマンドを実行<br/><br/><code style='color: white'>dd</code> - 行削除<br/><code style='color: white'>yy</code> - 行コピー<br/><code style='color: white'>p</code> - 貼り付け<br/><code style='color: white'>u</code> - 元に戻す"]

    Insert["✏️ インサートモード<br/>━━━━━━<br/>文字を入力する<br/><br/>普通のエディタと同じように<br/>タイピングできる"]

    Visual["📋 ビジュアルモード<br/>━━━━━━<br/>テキストを選択する<br/><br/><code style='color: white'>v</code> - 文字選択<br/><code style='color: white'>V</code> - 行選択"]

    Command["💻 コマンドモード<br/>━━━━━━<br/>ファイル操作<br/><br/><code style='color: white'>:w</code> - 保存<br/><code style='color: white'>:q</code> - 終了<br/><code style='color: white'>:wq</code> - 保存して終了"]

    Normal -->|"<code style='color: white'>i</code> - カーソル位置から<br/><code style='color: white'>a</code> - カーソル後ろから<br/><code style='color: white'>o</code> - 新しい行"| Insert
    Insert -->|"<code style='color: white'>Esc</code>"| Normal

    Normal -->|"<code style='color: white'>v</code> - 文字選択<br/><code style='color: white'>V</code> - 行選択"| Visual
    Visual -->|"<code style='color: white'>Esc</code>"| Normal

    Normal -->|"<code style='color: white'>:</code>"| Command
    Command -->|"<code style='color: white'>Enter</code> or <code style='color: white'>Esc</code>"| Normal

    Note["💡 ポイント<br/>━━━━━━<br/>・すべての道はノーマルモードに通ず<br/>・迷ったら <code style='color: white'>Esc</code> を連打<br/>・ノーマルモードが基準地点"]

    style Normal fill:#e3f2fd
    style Insert fill:#fff3e0
    style Visual fill:#e8f5e9
    style Command fill:#ffe0b2
    style Note fill:#fffde7
```

最初は戸惑いますが、慣れると**「書く」と「動かす」を分離**できることの快適さが分かります。

**公式サイト:** https://neovim.io/

---

### 4. LazyVim（Neovimの設定フレームワーク）

#### LazyVimとは？

**LazyVim**は、Neovimの **設定済みフレームワーク** です。

昔は `.vimrc` を何百行も書いて育てていましたが、今はLazyVimを使っています。

#### なぜLazyVimか？

| 特徴 | 説明 |
|------|------|
| **すぐ使える** | インストールするだけで実用的な環境 |
| **遅延ロード** | 必要になるまでプラグインを読み込まない（高速） |
| **モジュール式** | 必要な機能だけ有効化できる |
| **更新が安定** | 破壊的変更が少ない |

#### LazyVimに含まれる主要機能

- **LSP（Language Server Protocol）**: コード補完、定義ジャンプ
- **Treesitter**: シンタックスハイライト
- **Telescope**: ファイル検索、文字列検索
- **Git連携**: 差分表示、blame
- **ファイラー**: Neo-tree

#### 「全部を自分で作らない」という判断

以前は設定を一から書くのが楽しかった。

でも今は、**メンテナンスコスト**を考えて、土台は既存のものを使い、**カスタマイズは最小限**にしています。

これは「**過剰設計を避ける**」という仕事観とも一致しています。

##### カスタマイズレベルの比較

```mermaid
flowchart LR
    subgraph Full["🔧 フルスクラッチ<br/>━━━━━━<br/>全部自分で作る"]
        F1["⏱️ セットアップ時間<br/><code style='color: white'>数週間〜数ヶ月</code>"]
        F2["🎯 自由度<br/><code style='color: white'>100%</code>"]
        F3["💰 メンテナンスコスト<br/><code style='color: white'>高い</code><br/><br/>プラグイン更新で壊れる<br/>新しい機能は自分で実装"]
        F4["📚 学習コスト<br/><code style='color: white'>高い</code><br/><br/>Vimscript/Lua<br/>プラグイン仕様"]
    end

    subgraph Framework["📦 フレームワーク<br/>━━━━━━<br/>LazyVim等を使う"]
        W1["⏱️ セットアップ時間<br/><code style='color: white'>数時間</code>"]
        W2["🎯 自由度<br/><code style='color: white'>70-80%</code>"]
        W3["💰 メンテナンスコスト<br/><code style='color: white'>低い</code><br/><br/>フレームワークが更新<br/>カスタマイズのみ管理"]
        W4["📚 学習コスト<br/><code style='color: white'>中程度</code><br/><br/>フレームワークの思想<br/>カスタマイズ方法"]
    end

    subgraph Default["🎨 デフォルト<br/>━━━━━━<br/>そのまま使う"]
        D1["⏱️ セットアップ時間<br/><code style='color: white'>即座</code>"]
        D2["🎯 自由度<br/><code style='color: white'>0%</code>"]
        D3["💰 メンテナンスコスト<br/><code style='color: white'>なし</code><br/><br/>更新は自動<br/>設定ファイル不要"]
        D4["📚 学習コスト<br/><code style='color: white'>低い</code><br/><br/>基本操作のみ"]
    end

    Note["💡 僕の選択<br/>━━━━━━<br/><b>フレームワーク（LazyVim）</b><br/><br/>・実用まで早い<br/>・メンテナンス楽<br/>・カスタマイズも可能<br/>・仕事で使い続けられる"]

    Full -.->|"以前はこれ"| Framework
    Framework -.->|"今はこれ"| Note

    style Full fill:#ffebee
    style Framework fill:#e8f5e9
    style Default fill:#e3f2fd
    style Note fill:#fff3e0
```

**公式サイト:** https://www.lazyvim.org/

---

## 僕のdotfiles構成

```
~/dotfiles/
├── .config/
│   ├── nvim/           # Neovim設定（LazyVim）
│   │   └── lua/
│   │       └── plugins/  # カスタムプラグイン
│   └── wezterm/        # WezTerm設定
├── .tmux.conf          # tmux設定
├── .zshrc              # シェル設定
└── install.sh          # セットアップスクリプト
```

### 設定のポイント

1. **最小限のカスタマイズ**: デフォルトを尊重し、本当に必要なものだけ変える
2. **ポータビリティ**: 新しい環境でも動くように依存を減らす
3. **ドキュメント化**: なぜその設定にしたかをコメントで残す

---

## dotfilesに現れる仕事の流儀

ここまでツールの説明をしてきましたが、本当に伝えたいのは **「なぜこの構成なのか」** です。

### 速さより「戻れること」

- すぐ再現できる
- 他の環境でも使える
- 壊れたら戻せる
- 説明できる

僕のdotfilesは「最強」ではありません。
でも、**仕事で使い続けられる形**にはなっています。

### dotfilesを見れば、その人が分かる

- 何を面倒だと感じているか
- どこで妥協しているか
- 何を自動化し、何を残しているか
- 他人が触れることを想定しているか

派手さはないけど、**仕事の姿勢は確実に滲み出る**。

---

## まとめ

| ツール | 何をするもの | 僕が選んだ理由 |
|--------|--------------|----------------|
| WezTerm | ターミナル | 軽い、安定、Lua設定 |
| tmux | 画面分割・セッション管理 | 作業が途切れない安心感 |
| Neovim | テキスト編集 | 思考を止めずに手を動かせる |
| LazyVim | Neovimの土台 | メンテナンスコストを下げる |

dotfilesは、単なる設定ファイルではなく、**自分がどう考えて仕事をしているかの履歴**です。

---

## 始め方（初心者向け）

1. **まずはターミナルに慣れる**: 普段使いのターミナルでコマンドを打つ
2. **Vimtutor**: `vimtutor` コマンドで基本操作を学ぶ（30分）
3. **tmuxを触る**: `tmux` で起動、`Ctrl+b %` で分割
4. **LazyVimを試す**: 公式のインストール手順に従う

##### 初心者向け学習ロードマップ

```mermaid
flowchart TB
    Start["🎯 スタート<br/>━━━━━━<br/>開発環境を整えたい！"]

    Week1["📅 Week 1<br/>━━━━━━<br/>🖥️ ターミナルに慣れる<br/><br/><code style='color: white'>cd</code> - ディレクトリ移動<br/><code style='color: white'>ls</code> - ファイル一覧<br/><code style='color: white'>cat</code> - ファイル表示<br/><code style='color: white'>grep</code> - 検索"]

    Week2["📅 Week 2<br/>━━━━━━<br/>📝 Vimの基礎<br/><br/><code style='color: white'>vimtutor</code> で30分学習<br/>・<code style='color: white'>i</code> で入力、<code style='color: white'>Esc</code> で戻る<br/>・<code style='color: white'>:wq</code> で保存終了<br/>・簡単な編集を試す"]

    Week3["📅 Week 3<br/>━━━━━━<br/>📦 tmuxの基礎<br/><br/><code style='color: white'>tmux</code> - 起動<br/><code style='color: white'>Ctrl+b %</code> - 縦分割<br/><code style='color: white'>Ctrl+b d</code> - 離脱<br/><code style='color: white'>tmux attach</code> - 再接続"]

    Week4["📅 Week 4<br/>━━━━━━<br/>🚀 LazyVimを試す<br/><br/>公式手順でインストール<br/>・既存プロジェクトで使う<br/>・わからない操作は調べる<br/>・少しずつ慣れる"]

    Practice["🎓 実践フェーズ<br/>━━━━━━<br/>実際の仕事で使う<br/><br/>・小さいタスクから<br/>・エラーを恐れない<br/>・dotfilesをGit管理"]

    Master["✅ 習熟<br/>━━━━━━<br/>自分のスタイル確立<br/><br/>・必要なカスタマイズ追加<br/>・チームに共有<br/>・継続的に改善"]

    Start --> Week1
    Week1 --> Week2
    Week2 --> Week3
    Week3 --> Week4
    Week4 --> Practice
    Practice --> Master

    Note1["💡 焦らない<br/>━━━━━━<br/>一度に全部<br/>覚える必要なし"]
    Note2["💡 実践重視<br/>━━━━━━<br/>読むより<br/>触って覚える"]
    Note3["💡 挫折OK<br/>━━━━━━<br/>戻ってきたら<br/>また続ければいい"]

    Week1 -.-> Note1
    Week3 -.-> Note2
    Practice -.-> Note3

    style Start fill:#e3f2fd
    style Week1 fill:#fff3e0
    style Week2 fill:#fff3e0
    style Week3 fill:#fff3e0
    style Week4 fill:#fff3e0
    style Practice fill:#e8f5e9
    style Master fill:#c8e6c9
    style Note1 fill:#fffde7
    style Note2 fill:#fffde7
    style Note3 fill:#fffde7
```

いきなり全部を変える必要はありません。
**一つずつ、必要になったら足していく**のがおすすめです。

---

## 参考リンク

- [Neovim公式](https://neovim.io/)
- [LazyVim公式](https://www.lazyvim.org/)
- [tmux公式](https://github.com/tmux/tmux)
- [WezTerm公式](https://wezfurlong.org/wezterm/)
- [Vim日本語ドキュメント](https://vim-jp.org/vimdoc-ja/)
