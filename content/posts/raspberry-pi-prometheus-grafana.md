---
title: "Raspberry Piで構築するPrometheus + Grafana監視基盤"
date: 2025-12-13
draft: false
tags: ["RaspberryPi", "Prometheus", "Grafana", "監視", "インフラ", "Docker"]
description: "自宅マルチサーバー環境を一元監視するPrometheus + Grafana基盤の構築記録"
cover:
  image: "images/covers/monitoring.jpg"
  alt: "Prometheus + Grafana監視基盤"
  relative: false
---

## はじめに

自宅のマルチサーバー環境を一元監視するため、Raspberry Pi上にPrometheus + Grafanaベースの監視基盤を構築しました。本記事では、設計思想から実装詳細、運用で得られた知見までを共有します。

## プロジェクト概要

| 項目 | 内容 |
|------|------|
| 目的 | 複数サーバーの統合監視・可視化 |
| 監視サーバー | Raspberry Pi 4 |
| 監視対象 | Webサーバー、DBサーバー、監視サーバー自身 |
| 使用技術 | Prometheus, Grafana, Docker, systemd, Tailscale |
| 開発期間 | 設計・構築・運用テストまで約1週間 |

## 技術スタック

### コアコンポーネント

- **Prometheus** - 時系列データベース・メトリクス収集
- **Grafana** - 可視化・ダッシュボード
- **Docker** - Exporter群のコンテナ管理
- **systemd** - サービス管理・自動起動

### Exporter（メトリクス収集エージェント）

| Exporter | 用途 | 収集メトリクス |
|----------|------|----------------|
| Node Exporter | システム監視 | CPU、メモリ、ディスク、ネットワーク |
| MySQL Exporter | MySQL/MariaDB監視 | クエリ数、接続数、レプリケーション |
| PostgreSQL Exporter | PostgreSQL監視 | 接続数、トランザクション、テーブル統計 |
| Nginx Exporter | Webサーバー監視 | リクエスト数、アクティブ接続数 |

### ネットワーク

- **Tailscale** - リモートサーバーとのセキュアな接続（VPN）
- **ローカルLAN** - 同一ネットワーク内サーバー監視

## システムアーキテクチャ

```text
┌─────────────────────────────────────────────────────────┐
│              監視サーバー (Raspberry Pi)                 │
│                                                         │
│   ┌─────────────────┐       ┌─────────────────┐        │
│   │   Prometheus    │──────▶│    Grafana      │        │
│   │   (TSDB)        │PromQL │  (Visualization)│        │
│   │   Port: 9090    │       │   Port: 3000    │        │
│   └────────┬────────┘       └─────────────────┘        │
│            │                                            │
│            │ HTTP Scrape (15秒間隔)                     │
│            │                                            │
│   ┌────────┴────────┐                                   │
│   │  Node Exporter  │  ← ローカルシステムメトリクス      │
│   │  :9100          │                                   │
│   ├─────────────────┤                                   │
│   │ Postgres Export │  ← Docker経由                     │
│   │  :9187          │                                   │
│   ├─────────────────┤                                   │
│   │  Nginx Exporter │  ← Docker経由                     │
│   └─────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
            │
            │ Tailscale VPN / LAN
            │
    ┌───────┴───────┬───────────────────┐
    │               │                   │
    ▼               ▼                   ▼
┌─────────┐   ┌─────────┐         ┌─────────┐
│  Web    │   │   DB    │         │  Other  │
│ Server  │   │ Server  │         │ Servers │
│ :9100   │   │ :9100   │         │ :9100   │
│         │   │ :9104   │         │         │
└─────────┘   └─────────┘         └─────────┘
```

## 実装のポイント

### 1. Hybrid運用（systemd + Docker）

メインサービス（Prometheus、Grafana）はsystemdで管理し、Exporterは用途に応じてsystemdとDockerを使い分けました。

**systemd管理のメリット:**
- OS起動時の自動起動が確実
- ログ管理がjournalctlに統合
- リソース消費が軽量

**Docker管理のメリット:**
- 環境依存なく同一設定でデプロイ可能
- バージョン管理が容易
- 設定変更時の切り戻しが簡単

```yaml
# docker-compose.yml（Exporter群）
services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      - DATA_SOURCE_NAME=postgresql://user:pass@db:5432/postgres
    restart: unless-stopped
```

### 2. 自動セットアップスクリプト

新規サーバーへの監視エージェント導入を自動化するスクリプトを作成しました。

```bash
#!/bin/bash
# setup-db-server.sh - DBサーバー用

# アーキテクチャ自動判定（arm64/amd64対応）
ARCH=$(uname -m)
if [ "$ARCH" = "aarch64" ]; then
    ARCH_SUFFIX="arm64"
else
    ARCH_SUFFIX="amd64"
fi

# Node Exporterインストール
wget -q "https://github.com/.../node_exporter-${VERSION}.linux-${ARCH_SUFFIX}.tar.gz"

# DB種別自動判定
if command -v mysql &> /dev/null; then
    install_mysql_exporter
fi
if command -v psql &> /dev/null; then
    install_postgres_exporter
fi
```

**特徴:**
- ARM64/AMD64両対応
- インストール済みDBを自動検出
- 監視ユーザー作成・権限設定を自動化
- 完了後にPrometheus設定の追記内容を出力

### 3. Tailscaleによるリモート監視

異なるネットワークにあるWebサーバーをTailscale経由で監視しています。

```yaml
# prometheus.yml
- job_name: 'web-server'
  static_configs:
    - targets: ['100.126.77.58:9100']  # Tailscale IP
      labels:
        instance: 'web-ras'
```

**メリット:**
- ファイアウォール設定不要
- 暗号化通信
- NATを超えた接続

### 4. Grafana Provisioning

データソースとダッシュボードを設定ファイルで管理し、再構築時も同一環境を再現可能にしました。

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
```

## 運用実績

### 稼働状況

| 項目 | 数値 |
|------|------|
| 連続稼働時間 | 2週間以上 |
| メトリクス収集間隔 | 15秒 |
| データ保持期間 | 15日（デフォルト） |
| 監視対象サーバー | 3台 |

### 監視対象メトリクス数

```bash
# Prometheusのメトリクス数確認
$ curl -s localhost:9090/api/v1/label/__name__/values | jq '.data | length'
1,247
```

## 工夫した点

### 1. 最小権限の原則

DB監視ユーザーには必要最小限の権限のみ付与:

```sql
-- MySQL
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';

-- PostgreSQL
GRANT pg_monitor TO postgres_exporter;
```

### 2. セキュリティ設計

- Exporterポートはファイアウォールで監視サーバーのみ許可
- 認証情報ファイルは `chmod 600` で保護
- Tailscale経由でのリモート監視（ポート公開不要）

### 3. 拡張性

新規サーバー追加時の手順を標準化:

1. セットアップスクリプト実行（対象サーバー）
2. prometheus.ymlにターゲット追加（監視サーバー）
3. Grafanaダッシュボードインポート

## 今後の改善予定

- Alertmanagerによるアラート通知（Slack/Discord連携）
- Loki + Promtailによるログ収集統合
- Grafana Cloudへのメトリクス転送（長期保存）
- Kubernetes環境への拡張

## 得られた知見

### Raspberry Piでの運用

- **メモリ**: 4GB RAMで十分運用可能（Prometheus + Grafana + Docker Exporter）
- **ストレージ**: SDカードでも動作するが、SSDを推奨（書き込み耐久性）
- **ARM64対応**: 主要なExporterはすべてARM64ビルドが提供されている

### Prometheusの設計思想

Pull型アーキテクチャにより:
- 監視対象がダウンしても監視サーバーに影響なし
- ファイアウォール設定が単純（監視サーバー→対象への一方向）
- スケールアウトが容易

## まとめ

Raspberry PiとOSSツールのみで、商用監視サービスに匹敵する監視基盤を構築できました。

**このプロジェクトで使用した技術:**
- インフラ: Linux, systemd, Docker, Docker Compose
- 監視: Prometheus, Grafana, 各種Exporter
- ネットワーク: Tailscale, ファイアウォール設定
- 自動化: Bashスクリプト

**対応可能な業務:**
- オンプレミス/クラウド環境の監視基盤構築
- Prometheus/Grafanaの導入・運用支援
- インフラの可視化・メトリクス設計
- アラート設計・通知設定
