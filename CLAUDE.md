# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

フリーランスエンジニアのポートフォリオ・ブログサイト。Hugoで構築し、GitHub Pagesでホスティング。

## コマンド

```bash
# ローカル開発サーバー起動（下書き含む）
hugo server -D

# 本番ビルド
hugo --minify

# 新規記事作成
hugo new posts/記事名.md
```

## アーキテクチャ

- **フレームワーク**: Hugo (静的サイトジェネレーター)
- **テーマ**: PaperMod (Git submodule)
- **デプロイ**: GitHub Actions → GitHub Pages

## ディレクトリ構成

```
content/
├── posts/     # ブログ記事（Markdown）
└── about/     # Aboutページ
```

## 記事のフロントマター

```yaml
---
title: "記事タイトル"
date: 2024-12-13
draft: false          # true で下書き
tags: ["タグ1", "タグ2"]
description: "記事の説明"
---
```

## デプロイ

`main`ブランチにpushすると、GitHub Actionsが自動でビルド・デプロイを実行。

## 設定ファイル

- `hugo.toml`: サイト設定（タイトル、メニュー、テーマ設定など）
- `.github/workflows/deploy.yml`: CI/CDパイプライン
