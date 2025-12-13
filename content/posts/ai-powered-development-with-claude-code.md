---
title: "AIで爆速開発：コンテキストウィンドウを理解し、Obsidian×Claude Codeで並行開発する技術"
date: 2025-12-13
draft: false
tags: ["AI", "Claude Code", "Obsidian", "プロンプトエンジニアリング", "開発効率化", "LLM", "実務"]
description: "コンテキストウィンドウの本質を理解し、Obsidianで再利用可能なプロンプトを設計。Claude Codeで複数タスクを並行開発する爆速エンジニアになるための完全ガイド"
cover:
  image: "images/covers/ai-development.jpg"
  alt: "AI爆速開発"
  relative: false
---

## この記事の対象読者

- AIを使っているけど、思ったほど速くならない人
- ChatGPTやClaudeに長文を投げて「忘れられた」経験がある人
- プロンプトを毎回ゼロから書いている人
- Claude Codeを使い始めたけど、まだ活用しきれていない人

この記事では、**コンテキストウィンドウの本質**を理解し、**Obsidianでプロンプトを資産化**し、**Claude Codeで並行開発**することで、開発速度を劇的に上げる方法を解説します。

---

## なぜAIを使っても速くならないのか

### よくある失敗パターン

```
❌ パターン1：コンテキスト溢れ
「さっき言ったこと覚えてる？」→「申し訳ありませんが...」

❌ パターン2：毎回ゼロからプロンプト
同じような指示を毎回手打ち → 時間の無駄

❌ パターン3：直列作業
1つのタスクが終わるまで次に進めない → 待ち時間

❌ パターン4：曖昧な指示
「いい感じにして」→ 期待と違う結果 → やり直し
```

### この記事で得られること

```
✅ コンテキストウィンドウを意識した効率的な対話
✅ 再利用可能なプロンプトテンプレートの設計
✅ 複数タスクの並行開発ワークフロー
✅ Claude Codeの真の力を引き出す使い方
```

---

# 第1部：コンテキストウィンドウを理解する

## コンテキストウィンドウとは

**コンテキストウィンドウ** は、LLMが一度に「見える」テキストの量です。

```
┌─────────────────────────────────────────────────────────────┐
│                  コンテキストウィンドウ                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ システムプロンプト（CLAUDE.md等）                      │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │ 会話履歴（過去のやり取り）                            │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │ 現在のユーザー入力                                   │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │ ファイルの内容（読み込んだコード等）                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ← これを超えると「忘れる」→                                │
└─────────────────────────────────────────────────────────────┘
```

## 主要LLMのコンテキストウィンドウ

| モデル | コンテキスト長 | 日本語文字数目安 |
|--------|--------------|----------------|
| Claude 3.5 Sonnet | 200K tokens | 約15万文字 |
| Claude 3 Opus | 200K tokens | 約15万文字 |
| GPT-4 Turbo | 128K tokens | 約10万文字 |
| GPT-4o | 128K tokens | 約10万文字 |
| Gemini 1.5 Pro | 1M tokens | 約75万文字 |

**注意：** 大きければいいわけではない。後述する「Lost in the Middle」問題があります。

## コンテキストウィンドウの消費

```
1回の会話で消費されるトークン：

システムプロンプト:     ~2,000 tokens
会話履歴（10往復）:    ~10,000 tokens
読み込んだファイル:    ~50,000 tokens
ツール呼び出し結果:    ~20,000 tokens
─────────────────────────────────
合計:                  ~82,000 tokens
```

## Lost in the Middle問題

LLMは **最初と最後の情報を重視** し、**中間の情報を忘れやすい** 傾向があります。

```
┌──────────────────────────────────────────────────────┐
│  重要度                                              │
│    ▲                                                 │
│  高│ ██                                      ██     │
│    │ ████                                  ████     │
│    │ ██████                              ██████     │
│    │ ████████                          ████████     │
│  低│ ██████████████████████████████████████████     │
│    └──────────────────────────────────────────→     │
│      最初        ← 中間 →          最後             │
└──────────────────────────────────────────────────────┘
```

### 対策

```
1. 重要な指示は最初か最後に置く
2. 長い会話は要約して新しいセッションに引き継ぐ
3. 関連情報をまとめて、散らばらせない
4. 不要な情報は含めない（ノイズを減らす）
```

---

## コンテキストを意識した対話術

### 悪い例：コンテキストを無駄遣い

```markdown
# ユーザーの指示（悪い例）

えーっと、昨日から考えてたんですけど、
あのですね、ちょっと難しいかもしれないんですが、
ユーザー登録の機能を作りたくて、
でも、どうしようかなと思ってて、
あ、言い忘れましたが、フレームワークはNext.jsで、
データベースはPostgreSQLを使ってます。
それで、認証はNextAuthを使いたいんですよね。
あと、メール確認機能も欲しいかな...
```

### 良い例：構造化された指示

```markdown
# ユーザー登録機能の実装

## 環境
- Next.js 14 (App Router)
- PostgreSQL + Prisma
- NextAuth.js

## 要件
1. メールアドレスとパスワードで登録
2. メール確認機能
3. パスワードハッシュ化（bcrypt）

## 期待するファイル構成
- `app/api/auth/register/route.ts`
- `prisma/schema.prisma` への追加
- `lib/mail.ts`（メール送信）
```

### 効果

| 項目 | 悪い例 | 良い例 |
|------|--------|--------|
| トークン消費 | 多い | 少ない |
| AIの理解度 | 曖昧 | 明確 |
| 出力の質 | ブレる | 安定 |
| やり直し回数 | 多い | 少ない |

---

## 長いセッションの管理

### セッションを分割する基準

```
□ 10往復以上の会話
□ 別のタスクに移る時
□ 大きなファイルを複数読み込んだ後
□ 「さっき言ったこと」を参照し始めた時
```

### 引き継ぎプロンプト

```markdown
# セッション引き継ぎ

## 前回のセッションで完了したこと
- ユーザー認証機能の実装
- Prismaスキーマの定義
- メール送信機能

## 今回のセッションでやること
- パスワードリセット機能
- アカウント設定ページ

## 重要な決定事項
- 認証方式: JWT + HttpOnly Cookie
- メール: SendGrid
- パスワードハッシュ: bcrypt (12 rounds)
```

---

# 第2部：Obsidianでプロンプトを資産化する

## なぜObsidianか

| 特徴 | メリット |
|------|---------|
| Markdownベース | そのままコピペできる |
| ローカルファイル | Git管理できる |
| リンク機能 | プロンプト間を参照できる |
| テンプレート | 変数を埋め込める |
| 検索が速い | 必要なプロンプトをすぐ見つける |

## プロンプト管理のディレクトリ構成

```
prompts/
├── _templates/           # テンプレート
│   ├── code-review.md
│   ├── feature-impl.md
│   ├── bug-fix.md
│   └── refactor.md
├── _variables/           # 変数定義
│   ├── tech-stack.md
│   ├── coding-style.md
│   └── project-context.md
├── _components/          # 再利用可能なパーツ
│   ├── error-handling.md
│   ├── testing-policy.md
│   └── security-checklist.md
└── sessions/             # セッションログ
    ├── 2025-12-13-auth.md
    └── 2025-12-13-api.md
```

---

## 変数定義ファイル

### `_variables/tech-stack.md`

```markdown
# 技術スタック変数

## フロントエンド
- **フレームワーク**: Next.js 14 (App Router)
- **言語**: TypeScript 5.3
- **スタイリング**: Tailwind CSS 3.4
- **状態管理**: Zustand
- **フォーム**: React Hook Form + Zod

## バックエンド
- **ランタイム**: Node.js 20 LTS
- **フレームワーク**: Hono（またはExpress）
- **ORM**: Prisma 5
- **DB**: PostgreSQL 16

## インフラ
- **ホスティング**: Vercel
- **DB**: Supabase
- **認証**: NextAuth.js v5

## 開発ツール
- **パッケージマネージャ**: pnpm
- **リンター**: ESLint + Prettier
- **テスト**: Vitest + Playwright
```

### `_variables/coding-style.md`

```markdown
# コーディングスタイル変数

## 命名規則
- **変数・関数**: camelCase
- **コンポーネント**: PascalCase
- **定数**: UPPER_SNAKE_CASE
- **ファイル名**: kebab-case

## TypeScript
- `any` 禁止（型を明示する）
- `interface` より `type` を優先
- 関数の戻り値は明示する

## React
- 関数コンポーネントのみ
- Hooksは `use` プレフィックス
- Props は分割代入で受け取る

## エラーハンドリング
- try-catch は最小限のスコープ
- エラーは適切にログ出力
- ユーザー向けメッセージは別途定義

## コメント
- 「なぜ」を書く（「何」はコードで分かる）
- JSDocは公開APIにのみ
- TODOには担当者と期限を
```

### `_variables/project-context.md`

```markdown
# プロジェクトコンテキスト

## プロジェクト概要
ECサイトのバックエンド API

## ビジネス要件
- 商品カタログ管理
- カート・注文処理
- ユーザー認証・認可
- 決済連携（Stripe）

## 現在のフェーズ
MVP開発（3ヶ月目）

## 重要な制約
- レスポンス 200ms 以内
- 同時ユーザー 1000人想定
- GDPR準拠必須

## チーム構成
- バックエンド: 2名
- フロント: 2名
- デザイナー: 1名
```

---

## プロンプトテンプレート

### `_templates/feature-impl.md`

```markdown
# 機能実装プロンプト

## 変数読み込み
{{embed:_variables/tech-stack.md}}
{{embed:_variables/coding-style.md}}

---

## 実装依頼

### 機能名
{{feature_name}}

### 概要
{{feature_description}}

### 要件
{{requirements}}

### API設計（該当する場合）
- エンドポイント: {{endpoint}}
- メソッド: {{method}}
- リクエスト: {{request_body}}
- レスポンス: {{response_body}}

### 期待する成果物
1. 実装コード
2. 単体テスト
3. 使用例（必要に応じて）

### 制約・注意事項
{{constraints}}

---

## 出力形式
- ファイルパスを明示
- コードブロックで囲む
- 重要な設計判断は説明を添える
```

### `_templates/code-review.md`

```markdown
# コードレビュープロンプト

## 変数読み込み
{{embed:_variables/coding-style.md}}

---

## レビュー対象

```{{language}}
{{code}}
```

## レビュー観点

### 必須チェック
- [ ] セキュリティ脆弱性
- [ ] パフォーマンス問題
- [ ] エラーハンドリング
- [ ] エッジケース

### 品質チェック
- [ ] 可読性
- [ ] 保守性
- [ ] テスタビリティ
- [ ] コーディング規約準拠

## 出力形式

```markdown
## 重大な問題（修正必須）
-

## 改善提案
-

## 良い点
-

## 修正後のコード（必要な場合）
```
```

### `_templates/bug-fix.md`

```markdown
# バグ修正プロンプト

## 変数読み込み
{{embed:_variables/tech-stack.md}}

---

## バグ報告

### 現象
{{symptom}}

### 再現手順
{{steps}}

### 期待する動作
{{expected}}

### 実際の動作
{{actual}}

### 関連コード

```{{language}}
{{code}}
```

### エラーログ（ある場合）

```
{{error_log}}
```

---

## 依頼内容
1. 原因の特定
2. 修正方法の提案
3. 修正コード
4. 再発防止策

## 出力形式
- 原因を簡潔に説明
- 修正コードをファイルパス付きで提示
- テストケースを追加（可能なら）
```

---

## Obsidianテンプレートプラグインの活用

### Templaterプラグインの設定

```javascript
// Templater スクリプト例
// prompts/_scripts/fill-template.js

module.exports = async (tp) => {
    // 変数ファイルを読み込む
    const techStack = await tp.file.include("[[_variables/tech-stack]]");
    const codingStyle = await tp.file.include("[[_variables/coding-style]]");

    // ユーザー入力を取得
    const featureName = await tp.system.prompt("機能名を入力:");
    const description = await tp.system.prompt("機能の説明:");

    return `
# ${featureName} 実装依頼

## 技術スタック
${techStack}

## コーディングスタイル
${codingStyle}

## 機能説明
${description}

## 要件
-

## 制約
-
`;
};
```

### 使用例

```markdown
1. Obsidianで新規ノートを作成
2. Templater: "Insert Template" を実行
3. feature-impl テンプレートを選択
4. プロンプトに変数が展開される
5. Claude Codeにコピペ
```

---

# 第3部：Claude Codeで並行開発

## Claude Codeとは

**Claude Code** は、Anthropicの公式CLIツールです。

```bash
# インストール
npm install -g @anthropic-ai/claude-code

# 起動
claude

# プロジェクトディレクトリで起動
cd my-project && claude
```

## Claude Codeの強み

| 特徴 | 説明 |
|------|------|
| ファイルアクセス | プロジェクトのコードを直接読み書き |
| コマンド実行 | npm, git, テスト等を実行 |
| 複数ファイル編集 | 一度に複数ファイルを変更 |
| コンテキスト維持 | CLAUDE.md でプロジェクト設定を永続化 |
| ツール呼び出し | 必要に応じて自動でツールを使用 |

## CLAUDE.mdでプロジェクト設定

```markdown
# CLAUDE.md

## プロジェクト概要
ECサイトのバックエンドAPI

## 技術スタック
- Node.js 20 + TypeScript
- Hono（Webフレームワーク）
- Prisma + PostgreSQL
- Vitest（テスト）

## ディレクトリ構成
```
src/
├── routes/      # APIエンドポイント
├── services/    # ビジネスロジック
├── repositories/# データアクセス
├── middleware/  # ミドルウェア
└── utils/       # ユーティリティ
```

## コマンド
```bash
pnpm dev        # 開発サーバー
pnpm test       # テスト実行
pnpm build      # ビルド
pnpm db:migrate # マイグレーション
```

## コーディング規約
- 関数は単一責任
- エラーは Result 型で返す
- テストカバレッジ 80% 以上

## 禁止事項
- any 型の使用
- console.log（本番コード）
- コミット前のテスト未実行
```

---

## 並行開発のワークフロー

### シングルタスク vs 並行開発

```
【シングルタスク】
タスクA ───────────────→ 完了
                         タスクB ───────────────→ 完了
                                                  タスクC ─────→
⏱️ 合計時間: A + B + C

【並行開発】
タスクA ───────────────→ 完了
タスクB ───────────────→ 完了
タスクC ───────────────→ 完了
⏱️ 合計時間: max(A, B, C)
```

### 並行開発が可能なケース

```
✅ 独立した機能の実装
   - ユーザー認証 と 商品カタログ
   - API実装 と フロントエンド

✅ 異なるレイヤーの作業
   - DBスキーマ設計 と UI設計
   - バックエンド と テスト作成

✅ リファクタリングと新機能
   - 既存コードの整理 と 新機能追加
```

### 並行開発が難しいケース

```
❌ 依存関係がある作業
   - 認証完了後にしか作れない保護リソース
   - DBスキーマ確定後のRepository実装

❌ 同じファイルを触る作業
   - コンフリクトの原因
```

---

## 複数ターミナルでの並行開発

### ターミナル構成

```
┌──────────────────────────────────────────────────────────┐
│ ターミナル1: Claude Code（機能A）                         │
│ $ claude                                                 │
│ > ユーザー認証機能を実装して                              │
├──────────────────────────────────────────────────────────┤
│ ターミナル2: Claude Code（機能B）                         │
│ $ claude                                                 │
│ > 商品検索APIを実装して                                   │
├──────────────────────────────────────────────────────────┤
│ ターミナル3: 通常のシェル（監視・確認）                    │
│ $ pnpm dev                                               │
│ $ git status                                             │
└──────────────────────────────────────────────────────────┘
```

### tmuxでの構成

```bash
# tmuxセッション作成
tmux new-session -s dev

# ウィンドウ分割
# Ctrl+b % (縦分割)
# Ctrl+b " (横分割)

# 各ペインでClaude Codeを起動
# ペイン1: claude（認証機能）
# ペイン2: claude（商品機能）
# ペイン3: 開発サーバー + git
```

---

## 実践的なプロンプトパターン

### パターン1：機能実装

```markdown
# 商品検索API実装

## 要件
- GET /api/products/search
- クエリパラメータ: q（検索語）, category, minPrice, maxPrice
- ページネーション対応（limit, offset）
- レスポンス: 商品リスト + 総件数

## 技術制約
- Honoのルーティングを使用
- Prismaでクエリ
- Zodでバリデーション

## 実装してほしいファイル
1. src/routes/products.ts（ルート定義）
2. src/services/product-search.ts（ビジネスロジック）
3. src/repositories/product.ts（データアクセス）
4. tests/product-search.test.ts（テスト）
```

### パターン2：バグ修正

```markdown
# バグ修正依頼

## 現象
注文確定時に500エラーが発生する

## 再現条件
- カートに3つ以上の商品がある
- クーポンコードを適用した状態

## エラーログ
```
TypeError: Cannot read property 'discount' of undefined
    at calculateTotal (src/services/order.ts:45)
```

## 関連ファイル
- src/services/order.ts
- src/services/coupon.ts

## 依頼
1. 原因を特定して
2. 修正コードを提示して
3. 同様のバグを防ぐテストを追加して
```

### パターン3：リファクタリング

```markdown
# リファクタリング依頼

## 対象
src/services/payment.ts（300行）

## 問題点
- 1つの関数が100行以上
- 条件分岐が深くネスト
- テストが書きにくい

## 期待する構造
- 決済プロバイダーごとにStrategyパターン
- 各関数20行以内
- 依存性注入でテスタブルに

## 制約
- 外部APIの呼び出しは変えない
- 既存のテストは通るように
```

### パターン4：テスト追加

```markdown
# テスト追加依頼

## 対象
src/services/cart.ts

## 現状
テストなし

## 依頼
1. 単体テストを追加（Vitest）
2. カバレッジ80%以上
3. エッジケースを含める
   - 空のカート
   - 在庫切れ商品
   - 最大数量超過
   - クーポン適用

## テストファイル
tests/cart.test.ts
```

---

## 効率を上げるテクニック

### 1. 「続き」で再開

```markdown
# セッション途中で中断した場合

## 前回の作業
- src/routes/users.ts を作成中
- 認証ミドルウェアまで完成
- プロフィール更新APIが未完了

## 続き
プロフィール更新APIを完成させて
```

### 2. 差分だけ指示

```markdown
# 既存コードへの追加

## 現在の状態
src/routes/products.ts に GET /products が実装済み

## 追加依頼
同じファイルに以下を追加:
- POST /products（新規作成）
- PUT /products/:id（更新）
- DELETE /products/:id（削除）

## 注意
既存のGETは変更しないで
```

### 3. 出力形式を指定

```markdown
## 出力形式

### コード
- ファイルパスをコメントで明示
- 変更部分のみ（全体を書き直さない）

### 説明
- 3行以内で簡潔に
- 重要な設計判断のみ

### やらないこと
- 既存コードの説明
- 基本的な文法の解説
```

### 4. TodoWriteを活用

```markdown
## 複数タスクの依頼

以下のタスクを順番に実行して。
各タスク完了時にTodoWriteで進捗を更新して。

1. [ ] Prismaスキーマ更新
2. [ ] マイグレーション実行
3. [ ] Repository実装
4. [ ] Service実装
5. [ ] Route実装
6. [ ] テスト作成
7. [ ] 動作確認
```

---

## 並行開発の実践例

### シナリオ：ECサイトの機能追加

```
【タスク分解】
A: ユーザーのお気に入り機能
B: 商品レビュー機能
C: 注文履歴のCSVエクスポート
```

### ターミナル1（お気に入り機能）

```markdown
# お気に入り機能の実装

## API
- POST /api/favorites/:productId（追加）
- DELETE /api/favorites/:productId（削除）
- GET /api/favorites（一覧取得）

## DBスキーマ
favorites テーブル: userId, productId, createdAt

## 実装してほしいもの
1. prisma/schema.prisma への追加
2. src/routes/favorites.ts
3. src/services/favorite.ts
4. tests/favorite.test.ts
```

### ターミナル2（レビュー機能）

```markdown
# 商品レビュー機能の実装

## API
- POST /api/products/:id/reviews（投稿）
- GET /api/products/:id/reviews（一覧）
- DELETE /api/reviews/:id（削除）

## DBスキーマ
reviews テーブル: userId, productId, rating, comment, createdAt

## 注意
- 同じ商品に複数レビュー禁止
- rating は 1-5 の整数

## 実装してほしいもの
1. prisma/schema.prisma への追加
2. src/routes/reviews.ts
3. src/services/review.ts
4. tests/review.test.ts
```

### ターミナル3（CSVエクスポート）

```markdown
# 注文履歴CSVエクスポート

## API
- GET /api/orders/export?from=2025-01-01&to=2025-12-31

## 仕様
- 認証必須（自分の注文のみ）
- Content-Type: text/csv
- ファイル名: orders_YYYYMMDD.csv

## CSVカラム
orderId, orderDate, productName, quantity, price, total

## 実装
1. src/routes/orders.ts に追加
2. src/services/order-export.ts
3. tests/order-export.test.ts
```

### マージ作業（並行作業後）

```bash
# 各ブランチの変更を確認
git diff main..feature/favorites
git diff main..feature/reviews
git diff main..feature/csv-export

# Prismaスキーマのマージ（手動調整が必要な場合あり）
# → 両方の変更を含むスキーマを作成

# マイグレーション
pnpm db:migrate

# テスト実行
pnpm test

# マージ
git checkout main
git merge feature/favorites
git merge feature/reviews
git merge feature/csv-export
```

---

## トラブルシューティング

### 問題1：コンテキストが溢れた

```markdown
## 症状
Claude Codeが前の指示を忘れている

## 対策
1. /clear でセッションをリセット
2. 必要な情報だけを再度伝える
3. CLAUDE.md に重要な情報を書く
```

### 問題2：並行作業でコンフリクト

```markdown
## 症状
同じファイルを編集してマージできない

## 対策
1. 事前にファイルの責任範囲を分ける
2. 共通部分は先に完成させる
3. コンフリクト時はClaude Codeに解決を依頼
```

### 問題3：出力が期待と違う

```markdown
## 症状
指示したのと違うコードが生成される

## 対策
1. 具体的な例を示す
2. 「〜しないで」という否定形で制約を追加
3. 出力形式を厳密に指定
```

---

## まとめ

### コンテキストウィンドウの理解

```
1. トークン数を意識する
2. 重要な情報は最初か最後に
3. 長いセッションは分割する
4. 不要な情報は含めない
```

### Obsidianでのプロンプト管理

```
1. 変数ファイルで技術スタックを定義
2. テンプレートで構造化
3. コンポーネントで再利用
4. セッションログで履歴管理
```

### Claude Codeでの並行開発

```
1. CLAUDE.md でプロジェクト設定
2. 独立したタスクに分解
3. 複数ターミナルで同時進行
4. 定期的にマージ・確認
```

### 爆速エンジニアの習慣

| 習慣 | 効果 |
|------|------|
| プロンプトを資産化 | 毎回考えなくていい |
| 構造化された指示 | 出力が安定 |
| 並行開発 | 待ち時間ゼロ |
| 適切なタスク分割 | 手戻り減少 |

---

## 参考リンク

- [Claude Code公式](https://claude.ai/code)
- [Obsidian公式](https://obsidian.md/)
- [Templaterプラグイン](https://github.com/SilentVoid13/Templater)
- [Anthropic Prompt Engineering](https://docs.anthropic.com/claude/docs/prompt-engineering)
- [Lost in the Middle論文](https://arxiv.org/abs/2307.03172)
