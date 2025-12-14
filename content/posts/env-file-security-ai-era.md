---
title: ".envファイルをセキュアに扱う完全ガイド：AI時代の秘密管理"
date: 2024-12-14
draft: false
tags: ["セキュリティ", "開発環境", "シークレット管理", "AI", "チーム開発"]
description: ".envファイルは便利だが危険も多い。AIコーディングツールへの漏洩、チーム間の権限管理、そしてより良い代替手段まで。現代の秘密管理を徹底解説する。"
cover:
  image: "https://images.unsplash.com/photo-1555949963-ff9fe0c870eb?w=800"
  alt: "セキュリティとコード"
  relative: false
---

```mermaid
flowchart TB
    subgraph Problem["⚠️ 従来の問題"]
        ENV[.env ファイル]
        GIT[Gitに誤コミット]
        AI[AIに読み込まれる]
        SHARE[安全でない共有]
    end

    subgraph Solution["✅ 解決策"]
        ENCRYPT[暗号化 dotenvx]
        MANAGER[シークレット管理ツール]
        DENY[AIアクセス制御]
        RBAC[権限管理 RBAC]
    end

    ENV --> GIT
    ENV --> AI
    ENV --> SHARE

    GIT -.->|対策| ENCRYPT
    AI -.->|対策| DENY
    SHARE -.->|対策| MANAGER
    MANAGER --> RBAC

    style Problem fill:#ffebee
    style Solution fill:#e8f5e9
```

## .envファイル、本当に安全？

開発者なら誰でも使っている `.env` ファイル。

```
DATABASE_URL=postgres://user:password@localhost:5432/mydb
API_KEY=sk-1234567890abcdef
SECRET_TOKEN=super-secret-value
```

便利だ。環境ごとに設定を分けられる。コードに直接書かなくていい。

でも、2024年だけで**2,300万件以上のシークレットがGitHubにコミットされた**という調査がある。そして今、新たな脅威が加わった。

**AIコーディングツールが、あなたの.envを読んでいるかもしれない。**

Copilot、Claude Code、Cursor。これらのツールはコードを理解するためにプロジェクト内のファイルを読み込む。`.gitignore` に入れていても、ローカルファイルは読める。そしてその内容は、クラウド上のLLMに送信される可能性がある。

今回は、.envファイルをセキュアに扱う方法を、以下の観点から徹底解説する：

1. **AIツールに読み込ませない方法**
2. **チーム内で権限を分けて管理する方法**
3. **.envを超える、より良い代替手段**

---

## まず知っておくべき：.envの根本的な問題

### 問題1：プレーンテキストである

.envファイルは**暗号化されていない**。ファイルシステムにアクセスできれば、誰でも読める。

- 開発マシンが侵害されたら？
- 共有サーバーに置いてしまったら？
- バックアップから漏洩したら？

### 問題2：アクセス制御がない

.envファイルには**誰が見ていいか**という概念がない。

- インターンも本番DBのパスワードが見える
- 退職者のアクセスを取り消すには、全シークレットを変更する必要がある
- 監査ログもない

### 問題3：ローテーションが困難

シークレットを定期的に変更したい。でも.envでは：

- すべての開発者の.envを更新する必要がある
- CI/CDの設定も更新する必要がある
- 手作業でミスが起きやすい

---

## AIコーディングツールから.envを守る

### Claude Codeの場合

Claude Codeは `.claude/settings.json` でファイルアクセスを制御できる。

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)",
      "Read(./.env.keys)"
    ]
  }
}
```

これを設定すると、Claude Codeは指定したファイルを**完全に見えなくなる**。

確認方法：
```
Claude に「.envファイルの中身を教えて」と聞いてみる
→ 正しく設定されていれば「アクセスできません」と返答される
```

### GitHub Copilotの場合

Copilot Business/Enterpriseプランでは、Content Exclusionsで除外設定ができる：

```yaml
# リポジトリ設定で指定
- "/.env"
- "/.env.*"
- "/secrets/**"
```

**重要な制限**：Copilot CLIやAgent modeはContent Exclusionsをサポートしていない。

### Cursorの場合

Cursorは `.cursorignore` ファイルをサポート：

```
# .cursorignore
.env
.env.*
secrets/
*.pem
*.key
```

また、`.cursorindexingignore` でインデックス対象から除外することもできる。

### 全ツール共通の対策

最も確実な方法は、**シークレットをプロジェクトディレクトリの外に置く**こと。

```bash
# プロジェクト外に配置
~/.secrets/myproject/.env

# シンボリックリンクは作らない（AIツールが辿る可能性）
# 環境変数で直接読み込む
source ~/.secrets/myproject/.env
```

---

## 解決策1：dotenvxで暗号化する

[dotenvx](https://dotenvx.com/) は、dotenvの作者が開発した「暗号化対応版」だ。

### 基本的な使い方

```bash
# インストール
npm install -g @dotenvx/dotenvx

# .envを作成
echo "API_KEY=sk-secret123" > .env

# 暗号化
dotenvx encrypt
```

暗号化後の.envファイル：

```
#/-------------------[DOTENV_PUBLIC_KEY]--------------------/
#/            public-key encryption for .env files          /
#/       [how it works](https://dotenvx.com/encryption)     /
#/----------------------------------------------------------/
DOTENV_PUBLIC_KEY="034af93e..."

# .env
API_KEY="encrypted:BEn9t8..."
```

### 鍵の管理

暗号化すると `.env.keys` ファイルが生成される：

```
DOTENV_PRIVATE_KEY="ec9e6..."
```

**重要**：
- `.env`（暗号化済み）→ **Gitにコミット可能**
- `.env.keys`（秘密鍵）→ **絶対にGitに入れない**

```bash
# .gitignoreに追加
echo ".env.keys" >> .gitignore
```

### 実行時の復号

```bash
# ローカル開発（.env.keysがある場合）
dotenvx run -- node app.js

# 本番環境（環境変数で秘密鍵を渡す）
DOTENV_PRIVATE_KEY="ec9e6..." dotenvx run -- node app.js
```

### 複数環境の管理

```bash
# 開発環境
dotenvx encrypt -f .env.development

# 本番環境
dotenvx encrypt -f .env.production

# CI環境
dotenvx encrypt -f .env.ci
```

各環境の秘密鍵は `.env.keys` にまとめて保存される：

```
DOTENV_PRIVATE_KEY_DEVELOPMENT="..."
DOTENV_PRIVATE_KEY_PRODUCTION="..."
DOTENV_PRIVATE_KEY_CI="..."
```

### dotenvxのメリット

1. **Gitにコミットできる**：暗号化されているので安全
2. **既存の.envワークフローを維持**：学習コスト低
3. **AIツールに読まれても大丈夫**：復号鍵がなければ無意味

---

## 解決策2：シークレット管理ツールを使う

チームで権限管理が必要な場合、専用ツールを検討すべきだ。

### 比較表

| ツール | 特徴 | 向いている組織 |
|--------|------|--------------|
| **Infisical** | OSS、セルフホスト可、モダンUI | 自社運用したいチーム |
| **Doppler** | SaaS、導入が簡単、DX重視 | スタートアップ、中小 |
| **HashiCorp Vault** | 高機能、複雑、エンタープライズ向け | 大企業、規制業界 |
| **AWS Secrets Manager** | AWS統合、マネージド | AWSメインのチーム |
| **1Password** | 開発者CLI、馴染みやすい | 既存1Passwordユーザー |

### Infisical：OSSで権限管理

[Infisical](https://infisical.com/) はオープンソースのシークレット管理プラットフォーム。

**特徴**：
- エンドツーエンド暗号化
- RBAC（ロールベースアクセス制御）
- バージョン管理・監査ログ
- セルフホスト可能

```bash
# CLIインストール
brew install infisical/get-cli/infisical

# ログイン
infisical login

# シークレット取得
infisical run -- node app.js
```

**権限設定例**：
- 開発者 → development環境のみアクセス可
- リーダー → development + staging
- SRE → すべての環境にアクセス可

### Doppler：SaaSで手軽に

[Doppler](https://www.doppler.com/) はDX重視のSaaS型シークレット管理。

```bash
# CLIインストール
brew install dopplerhq/cli/doppler

# セットアップ
doppler setup

# 実行（.envファイル不要）
doppler run -- node app.js
```

**便利な機能**：
- ダッシュボードでシークレットを管理
- 変更時に自動で環境に反映
- Slack通知、監査ログ

### どれを選ぶべきか

```
┌─────────────────────────────────────────────────────────┐
│ 個人開発 or 小規模チーム                                 │
│ → dotenvx（暗号化.env）で十分                            │
├─────────────────────────────────────────────────────────┤
│ 権限管理が必要 + セルフホストしたい                       │
│ → Infisical                                             │
├─────────────────────────────────────────────────────────┤
│ 権限管理が必要 + 運用を任せたい                          │
│ → Doppler                                               │
├─────────────────────────────────────────────────────────┤
│ 複雑な要件 + 規制対応が必要                              │
│ → HashiCorp Vault                                       │
└─────────────────────────────────────────────────────────┘
```

---

## 解決策3：CI/CDでシークレットを注入する

本番環境では、.envファイル自体を使わないアプローチもある。

### GitHub Actionsの例

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          npm run deploy
```

シークレットはGitHubのリポジトリ設定で管理。コードには一切現れない。

### Kubernetesの例

```yaml
# secret.yaml（このファイル自体はGitに入れない）
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_URL: cG9zdGdyZXM6Ly91c2VyOnBhc3NAaG9zdDo1NDMyL2Ri
```

External Secrets Operatorを使えば、Vault/AWSなどから自動同期もできる。

---

## チーム運用のベストプラクティス

### 1. 新メンバーのオンボーディング

```bash
# ❌ 悪い例：Slackでシークレットを送る
「.envファイル送ったよ〜」

# ✅ 良い例：シークレット管理ツール経由
「Infisicalに招待したから、CLIでpullして」
```

### 2. 退職者の対応

```bash
# ❌ 悪い例：何もしない（シークレット知ってるのに）

# ✅ 良い例
1. シークレット管理ツールからアクセス権を削除
2. 重要なシークレットをローテーション
3. 監査ログで最終アクセスを確認
```

### 3. 本番シークレットへのアクセス

```yaml
# Infisicalの権限設定例
roles:
  developer:
    environments: [development]
    permissions: [read]

  lead:
    environments: [development, staging]
    permissions: [read, write]

  sre:
    environments: [development, staging, production]
    permissions: [read, write, admin]
```

### 4. シークレットのローテーション

```bash
# 定期ローテーション（月1回など）
infisical secrets set API_KEY="new-key-value" --env=production

# アプリは自動で新しい値を取得
# → ダウンタイムなし
```

---

## 私の構成：実例

参考までに、私が実際に使っている構成を紹介する。

### ローカル開発

```
プロジェクト/
├── .env.example      # サンプル（Gitに入れる）
├── .env              # dotenvxで暗号化済み（Gitに入れる）
├── .env.keys         # 秘密鍵（Gitに入れない）
└── .claude/
    └── settings.json # AIアクセス制御
```

```json
// .claude/settings.json
{
  "permissions": {
    "deny": [
      "Read(./.env.keys)",
      "Read(./.env.local)",
      "Read(./secrets/**)"
    ]
  }
}
```

### チーム開発

- 開発環境のシークレット → dotenvxで暗号化、Gitで共有
- 本番環境のシークレット → Infisicalで管理、RBACで権限制御
- CI/CD → GitHub Secretsから注入

### ワークフロー

```bash
# 新しいシークレットを追加
dotenvx set NEW_API_KEY="value"

# 暗号化されて.envに保存
git add .env
git commit -m "Add NEW_API_KEY"

# チームメンバーはpullするだけ
# .env.keysを持っていれば復号できる
```

---

## まとめ

.envファイルは便利だが、そのままでは危険が多い。

### やるべきこと

1. **AIツールからのアクセスをブロック**
   - Claude Code: `.claude/settings.json` の deny 設定
   - Copilot: Content Exclusions
   - Cursor: `.cursorignore`

2. **暗号化する**
   - dotenvxで.envを暗号化
   - 暗号化済み.envはGitにコミット可能
   - 秘密鍵は別管理

3. **チームで権限管理が必要なら専用ツール**
   - Infisical（OSS、セルフホスト）
   - Doppler（SaaS、手軽）
   - Vault（エンタープライズ）

4. **本番では.envを使わない選択肢も**
   - CI/CDで環境変数を注入
   - シークレット管理ツールから直接取得

### 最低限これだけはやる

もし「全部は無理」という場合、最低限これだけはやってほしい：

```bash
# 1. .gitignoreに追加
echo ".env" >> .gitignore
echo ".env.*" >> .gitignore
echo "!.env.example" >> .gitignore

# 2. AIツールのアクセス制御
# （各ツールの設定ファイルで.envをdeny）

# 3. サンプルファイルを用意
cp .env .env.example
# .env.exampleの値をダミーに置き換え
```

シークレット漏洩は、起きてからでは遅い。今日から対策を始めよう。

---

## 参考リンク

- [dotenvx公式](https://dotenvx.com/)
- [Infisical - OSS Secrets Management](https://infisical.com/)
- [Doppler - Secrets Management](https://www.doppler.com/)
- [GitGuardian - Secrets Management Best Practices](https://blog.gitguardian.com/secure-your-secrets-with-env/)
- [Claude Code Settings - Anthropic Docs](https://docs.claude.com/en/docs/claude-code/settings)
- [GitHub Copilot Content Exclusions](https://docs.github.com/en/copilot/how-tos/configure-content-exclusion/exclude-content-from-copilot)
