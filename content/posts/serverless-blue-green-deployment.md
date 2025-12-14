---
title: "サーバーレスで激安開発：Lambdaエイリアスで実現するBlue-Greenデプロイ完全ガイド"
date: 2025-12-13
draft: false
categories: ["インフラ・運用"]
tags: ["AWS", "Lambda", "サーバーレス", "Blue-Green", "デプロイ", "コスト削減", "Terraform", "実務"]
description: "月額数百円から始めるサーバーレス開発。Lambdaのバージョン・エイリアスでdev/prodを分離し、安全なBlue-Greenデプロイを実現。UIを捨てて顧客が喜ぶ設計思想まで"
cover:
  image: "images/covers/serverless-lambda.jpg"
  alt: "サーバーレスBlue-Greenデプロイ"
  relative: false
---

## この記事の対象読者

- サーバー代を限りなくゼロに近づけたい人
- EC2やECSの運用に疲れた人
- 「デプロイが怖い」から解放されたい人
- 小さく始めて、スケールする設計を学びたい人

この記事では、**サーバーレスが激安な理由**、**Lambdaのバージョン・エイリアス管理**、**Blue-Greenデプロイの実装**、そして**「UIを捨てる」設計思想**まで解説します。

---

## サーバーレスが「激安」な理由

### 従量課金の本質

```
【従来のサーバー】
24時間365日稼働 = 24時間365日課金
使っていない深夜も、アクセスがない時間も課金

【サーバーレス】
リクエストがあった時だけ課金
0リクエスト = 0円
```

### コスト比較（月間10万リクエスト想定）

| 項目 | EC2 (t3.micro) | Lambda |
|------|---------------|--------|
| 基本料金 | 約$8.5/月 | $0 |
| リクエスト | - | $0.02（100万リクエストまで無料） |
| 実行時間 | - | 約$0.20（128MB×100ms×10万回） |
| **月額合計** | **約$8.5** | **約$0.22** |

**約40倍の差。**

### 無料枠がすごい

```
AWS Lambda 無料枠（毎月）:
- 100万リクエスト
- 40万GB秒の実行時間

これで足りるサービスは多い。
月額 $0 でAPIが動く。
```

---

## サーバーレスアーキテクチャの全体像

```
┌─────────────────────────────────────────────────────────────┐
│                        クライアント                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                              │
│                   (HTTPS エンドポイント)                     │
│                                                             │
│   /api/users  ─────→  Lambda (users)                       │
│   /api/orders ─────→  Lambda (orders)                      │
│   /api/products ───→  Lambda (products)                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Lambda Functions                        │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                    │
│  │ users   │  │ orders  │  │products │                    │
│  │ 128MB   │  │ 256MB   │  │ 128MB   │                    │
│  └────┬────┘  └────┬────┘  └────┬────┘                    │
└───────┼────────────┼────────────┼───────────────────────────┘
        │            │            │
        ▼            ▼            ▼
┌─────────────────────────────────────────────────────────────┐
│                    DynamoDB / Aurora Serverless             │
│                       (従量課金DB)                          │
└─────────────────────────────────────────────────────────────┘
```

---

# 第1部：Lambdaのバージョンとエイリアス

## なぜバージョン管理が必要か

```
問題：Lambda を直接更新すると...

デプロイ前: 正常に動作
    ↓ デプロイ
デプロイ後: バグがある！
    ↓
全ユーザーに影響 💀
戻し方が分からない 💀💀
```

**解決策: バージョンとエイリアス**

## バージョンとは

**バージョン** は、Lambdaコードの **スナップショット（不変のコピー）** です。

```
Lambda: my-function
├── $LATEST (常に最新、変更可能)
├── Version 1 (不変)
├── Version 2 (不変)
├── Version 3 (不変)
└── Version 4 (不変) ← 現在の最新
```

```bash
# バージョンを発行
aws lambda publish-version \
    --function-name my-function \
    --description "v1.2.0 - ユーザー認証機能追加"

# バージョン一覧
aws lambda list-versions-by-function \
    --function-name my-function
```

## エイリアスとは

**エイリアス** は、特定のバージョンを指す **ポインタ（名前付きの参照）** です。

```
Lambda: my-function
│
├── Alias: prod ────→ Version 3 (安定版)
├── Alias: dev  ────→ Version 4 (最新版)
└── Alias: staging ─→ Version 4

デプロイ時:
  1. Version 5 を作成
  2. dev エイリアスを Version 5 に向ける
  3. 検証OK後、prod を Version 5 に向ける
```

```bash
# エイリアスを作成
aws lambda create-alias \
    --function-name my-function \
    --name prod \
    --function-version 3 \
    --description "本番環境"

# エイリアスを更新（別バージョンに向ける）
aws lambda update-alias \
    --function-name my-function \
    --name prod \
    --function-version 4

# エイリアス一覧
aws lambda list-aliases \
    --function-name my-function
```

## エイリアスのARN

```
# $LATEST を直接呼び出し（非推奨）
arn:aws:lambda:ap-northeast-1:123456789:function:my-function

# バージョンを指定
arn:aws:lambda:ap-northeast-1:123456789:function:my-function:3

# エイリアスを指定（推奨）
arn:aws:lambda:ap-northeast-1:123456789:function:my-function:prod
arn:aws:lambda:ap-northeast-1:123456789:function:my-function:dev
```

**API Gateway からはエイリアスを呼ぶ。**

---

# 第2部：Blue-Greenデプロイ

## Blue-Greenデプロイとは

```
【従来のデプロイ】
旧バージョン ──(停止)──> 新バージョン
              ↓
         ダウンタイム発生

【Blue-Greenデプロイ】
Blue (現行) ────────────────> そのまま稼働
                    ↓ 切り替え
Green (新版) ───────────────> 新しいトラフィック

ダウンタイムなし、即座にロールバック可能
```

## Lambdaでの実現

```
【デプロイ前】
prod エイリアス ──→ Version 3 (Blue)
dev エイリアス  ──→ Version 4

【デプロイ】
1. 新しいコードをアップロード
2. Version 5 を発行 (Green)
3. dev を Version 5 に向ける
4. 検証
5. prod を Version 5 に向ける ← 本番切り替え

【ロールバック（問題発生時）】
prod を Version 3 に戻す ← 即座に復旧
```

## トラフィックシフト（カナリアリリース）

```bash
# prod エイリアスのトラフィックを分割
# 90% → Version 3（現行）
# 10% → Version 5（新版）

aws lambda update-alias \
    --function-name my-function \
    --name prod \
    --function-version 3 \
    --routing-config '{"AdditionalVersionWeights": {"5": 0.1}}'

# 問題なければ 100% 切り替え
aws lambda update-alias \
    --function-name my-function \
    --name prod \
    --function-version 5 \
    --routing-config '{}'
```

```
トラフィック配分:
┌────────────────────────────────────────────────────┐
│ ██████████████████████████████████████████ 90%     │ → Version 3
│ ████ 10%                                           │ → Version 5
└────────────────────────────────────────────────────┘
```

---

# 第3部：実践的な構成

## ディレクトリ構成

```
my-serverless-app/
├── terraform/              # インフラ定義
│   ├── main.tf
│   ├── lambda.tf
│   ├── api_gateway.tf
│   ├── dynamodb.tf
│   └── variables.tf
├── src/                    # Lambdaコード
│   ├── handlers/
│   │   ├── users.py
│   │   ├── orders.py
│   │   └── products.py
│   ├── lib/
│   │   ├── db.py
│   │   └── utils.py
│   └── requirements.txt
├── scripts/
│   ├── deploy.sh
│   ├── rollback.sh
│   └── promote.sh
├── tests/
│   └── ...
└── .github/
    └── workflows/
        └── deploy.yml
```

## Terraform でインフラ定義

### lambda.tf

```hcl
# Lambda関数
resource "aws_lambda_function" "api" {
  function_name = "${var.project_name}-api"
  role          = aws_iam_role.lambda_role.arn
  handler       = "handlers.main.handler"
  runtime       = "python3.11"

  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  memory_size = 128
  timeout     = 10

  environment {
    variables = {
      DYNAMODB_TABLE = aws_dynamodb_table.main.name
      ENVIRONMENT    = "production"
    }
  }

  # $LATEST は直接使わない
  publish = true

  lifecycle {
    ignore_changes = [
      # エイリアスで管理するため、これらの変更は無視
      filename,
      source_code_hash,
    ]
  }
}

# dev エイリアス（最新バージョンを指す）
resource "aws_lambda_alias" "dev" {
  name             = "dev"
  function_name    = aws_lambda_function.api.function_name
  function_version = aws_lambda_function.api.version

  lifecycle {
    ignore_changes = [function_version]
  }
}

# prod エイリアス（安定バージョンを指す）
resource "aws_lambda_alias" "prod" {
  name             = "prod"
  function_name    = aws_lambda_function.api.function_name
  function_version = aws_lambda_function.api.version

  lifecycle {
    # デプロイスクリプトで管理するため無視
    ignore_changes = [function_version, routing_config]
  }
}

# Lambda用IAMロール
resource "aws_iam_role" "lambda_role" {
  name = "${var.project_name}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "dynamodb_access" {
  name = "dynamodb-access"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ]
      Resource = aws_dynamodb_table.main.arn
    }]
  })
}
```

### api_gateway.tf

```hcl
# API Gateway (HTTP API - 安い)
resource "aws_apigatewayv2_api" "main" {
  name          = "${var.project_name}-api"
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins = ["*"]
    allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_headers = ["Content-Type", "Authorization"]
  }
}

# prod ステージ（prod エイリアスを使用）
resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.main.id
  name        = "prod"
  auto_deploy = true

  stage_variables = {
    lambdaAlias = "prod"
  }
}

# dev ステージ（dev エイリアスを使用）
resource "aws_apigatewayv2_stage" "dev" {
  api_id      = aws_apigatewayv2_api.main.id
  name        = "dev"
  auto_deploy = true

  stage_variables = {
    lambdaAlias = "dev"
  }
}

# Lambda統合（エイリアスを動的に参照）
resource "aws_apigatewayv2_integration" "lambda" {
  api_id             = aws_apigatewayv2_api.main.id
  integration_type   = "AWS_PROXY"
  integration_method = "POST"

  # ステージ変数でエイリアスを切り替え
  integration_uri = "arn:aws:lambda:${var.aws_region}:${data.aws_caller_identity.current.account_id}:function:${aws_lambda_function.api.function_name}:$${stageVariables.lambdaAlias}"
}

# ルート定義
resource "aws_apigatewayv2_route" "default" {
  api_id    = aws_apigatewayv2_api.main.id
  route_key = "$default"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

# Lambda実行権限（prod）
resource "aws_lambda_permission" "api_gateway_prod" {
  statement_id  = "AllowAPIGatewayProd"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api.function_name
  qualifier     = aws_lambda_alias.prod.name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.main.execution_arn}/*/*"
}

# Lambda実行権限（dev）
resource "aws_lambda_permission" "api_gateway_dev" {
  statement_id  = "AllowAPIGatewayDev"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.api.function_name
  qualifier     = aws_lambda_alias.dev.name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.main.execution_arn}/*/*"
}

# 出力
output "api_endpoint_prod" {
  value = "${aws_apigatewayv2_api.main.api_endpoint}/prod"
}

output "api_endpoint_dev" {
  value = "${aws_apigatewayv2_api.main.api_endpoint}/dev"
}
```

### dynamodb.tf

```hcl
# DynamoDB（従量課金モード = 激安）
resource "aws_dynamodb_table" "main" {
  name         = "${var.project_name}-table"
  billing_mode = "PAY_PER_REQUEST"  # オンデマンド課金

  hash_key  = "PK"
  range_key = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  # GSIが必要な場合
  global_secondary_index {
    name            = "GSI1"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "ALL"
  }

  attribute {
    name = "GSI1PK"
    type = "S"
  }

  attribute {
    name = "GSI1SK"
    type = "S"
  }

  tags = {
    Environment = "production"
  }
}
```

---

## デプロイスクリプト

### scripts/deploy.sh

```bash
#!/bin/bash
set -e

# 設定
FUNCTION_NAME="my-app-api"
ALIAS_DEV="dev"
ALIAS_PROD="prod"

echo "=== Deploying Lambda ==="

# 1. コードをパッケージング
echo "Packaging code..."
cd src
pip install -r requirements.txt -t ./package
cp -r handlers lib package/
cd package
zip -r ../../lambda.zip .
cd ../..

# 2. Lambda を更新
echo "Updating Lambda function..."
aws lambda update-function-code \
    --function-name $FUNCTION_NAME \
    --zip-file fileb://lambda.zip \
    --publish

# 3. 最新バージョンを取得
LATEST_VERSION=$(aws lambda list-versions-by-function \
    --function-name $FUNCTION_NAME \
    --query 'Versions[-1].Version' \
    --output text)

echo "Published version: $LATEST_VERSION"

# 4. dev エイリアスを更新
echo "Updating dev alias to version $LATEST_VERSION..."
aws lambda update-alias \
    --function-name $FUNCTION_NAME \
    --name $ALIAS_DEV \
    --function-version $LATEST_VERSION

echo "=== Deployment complete ==="
echo "dev endpoint is now on version $LATEST_VERSION"
echo ""
echo "To promote to prod, run:"
echo "  ./scripts/promote.sh $LATEST_VERSION"
```

### scripts/promote.sh

```bash
#!/bin/bash
set -e

FUNCTION_NAME="my-app-api"
ALIAS_PROD="prod"
VERSION=$1

if [ -z "$VERSION" ]; then
    echo "Usage: $0 <version>"
    echo "Example: $0 5"
    exit 1
fi

# 現在のprodバージョンを取得（ロールバック用）
CURRENT_PROD_VERSION=$(aws lambda get-alias \
    --function-name $FUNCTION_NAME \
    --name $ALIAS_PROD \
    --query 'FunctionVersion' \
    --output text)

echo "=== Promoting to Production ==="
echo "Current prod version: $CURRENT_PROD_VERSION"
echo "New version: $VERSION"
echo ""

read -p "Are you sure? (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
    echo "Aborted"
    exit 1
fi

# カナリアリリース（10%から開始）
echo "Starting canary release (10%)..."
aws lambda update-alias \
    --function-name $FUNCTION_NAME \
    --name $ALIAS_PROD \
    --function-version $CURRENT_PROD_VERSION \
    --routing-config "{\"AdditionalVersionWeights\": {\"$VERSION\": 0.1}}"

echo "Waiting 60 seconds for monitoring..."
sleep 60

# エラー率をチェック（CloudWatch）
# 簡易版：手動確認
read -p "Is the new version healthy? (yes/no): " HEALTHY

if [ "$HEALTHY" != "yes" ]; then
    echo "Rolling back..."
    aws lambda update-alias \
        --function-name $FUNCTION_NAME \
        --name $ALIAS_PROD \
        --function-version $CURRENT_PROD_VERSION \
        --routing-config '{}'
    echo "Rolled back to version $CURRENT_PROD_VERSION"
    exit 1
fi

# 100%に切り替え
echo "Promoting to 100%..."
aws lambda update-alias \
    --function-name $FUNCTION_NAME \
    --name $ALIAS_PROD \
    --function-version $VERSION \
    --routing-config '{}'

echo "=== Promotion complete ==="
echo "prod is now on version $VERSION"
echo ""
echo "To rollback, run:"
echo "  ./scripts/rollback.sh $CURRENT_PROD_VERSION"
```

### scripts/rollback.sh

```bash
#!/bin/bash
set -e

FUNCTION_NAME="my-app-api"
ALIAS_PROD="prod"
VERSION=$1

if [ -z "$VERSION" ]; then
    echo "Usage: $0 <version>"
    exit 1
fi

echo "=== Rolling back to version $VERSION ==="

aws lambda update-alias \
    --function-name $FUNCTION_NAME \
    --name $ALIAS_PROD \
    --function-version $VERSION \
    --routing-config '{}'

echo "=== Rollback complete ==="
```

---

## GitHub Actions CI/CD

### .github/workflows/deploy.yml

```yaml
name: Deploy Lambda

on:
  push:
    branches:
      - main
      - develop
  workflow_dispatch:
    inputs:
      promote_version:
        description: 'Version to promote to prod'
        required: false

env:
  AWS_REGION: ap-northeast-1
  FUNCTION_NAME: my-app-api

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r src/requirements.txt -t src/package
          cp -r src/handlers src/lib src/package/

      - name: Package Lambda
        run: |
          cd src/package
          zip -r ../../lambda.zip .

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy to Lambda
        id: deploy
        run: |
          # コード更新 & バージョン発行
          VERSION=$(aws lambda update-function-code \
            --function-name $FUNCTION_NAME \
            --zip-file fileb://lambda.zip \
            --publish \
            --query 'Version' \
            --output text)

          echo "version=$VERSION" >> $GITHUB_OUTPUT

          # dev エイリアスを更新
          aws lambda update-alias \
            --function-name $FUNCTION_NAME \
            --name dev \
            --function-version $VERSION

          echo "Deployed version $VERSION to dev"

      - name: Run integration tests
        run: |
          # dev エンドポイントでテスト
          curl -f https://xxx.execute-api.ap-northeast-1.amazonaws.com/dev/health

      - name: Notify
        run: |
          echo "Version ${{ steps.deploy.outputs.version }} deployed to dev"
          echo "To promote to prod, run promote workflow"

  promote:
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch' && github.event.inputs.promote_version != ''

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Promote to prod
        run: |
          VERSION=${{ github.event.inputs.promote_version }}

          # 現在のバージョンを保存
          CURRENT=$(aws lambda get-alias \
            --function-name $FUNCTION_NAME \
            --name prod \
            --query 'FunctionVersion' \
            --output text)

          echo "Promoting version $VERSION (current: $CURRENT)"

          # prod エイリアスを更新
          aws lambda update-alias \
            --function-name $FUNCTION_NAME \
            --name prod \
            --function-version $VERSION

          echo "Promoted to prod"
```

---

# 第4部：Lambda コード例

### handlers/main.py

```python
import json
import os
from lib.db import DynamoDB
from lib.utils import response, parse_body

db = DynamoDB(os.environ['DYNAMODB_TABLE'])

def handler(event, context):
    """
    メインハンドラー
    API Gatewayからのリクエストをルーティング
    """
    path = event.get('rawPath', '/')
    method = event.get('requestContext', {}).get('http', {}).get('method', 'GET')

    # ルーティング
    routes = {
        ('GET', '/health'): health_check,
        ('GET', '/users'): list_users,
        ('POST', '/users'): create_user,
        ('GET', '/users/{id}'): get_user,
        ('PUT', '/users/{id}'): update_user,
        ('DELETE', '/users/{id}'): delete_user,
    }

    # パスパラメータを抽出
    path_params = event.get('pathParameters', {}) or {}

    # ルート検索
    for (route_method, route_path), handler_func in routes.items():
        if method == route_method and match_path(path, route_path):
            try:
                return handler_func(event, path_params)
            except Exception as e:
                print(f"Error: {e}")
                return response(500, {'error': 'Internal server error'})

    return response(404, {'error': 'Not found'})

def match_path(actual: str, pattern: str) -> bool:
    """簡易パスマッチング"""
    actual_parts = actual.strip('/').split('/')
    pattern_parts = pattern.strip('/').split('/')

    if len(actual_parts) != len(pattern_parts):
        return False

    for a, p in zip(actual_parts, pattern_parts):
        if p.startswith('{') and p.endswith('}'):
            continue
        if a != p:
            return False

    return True

def health_check(event, params):
    """ヘルスチェック"""
    return response(200, {
        'status': 'healthy',
        'version': os.environ.get('AWS_LAMBDA_FUNCTION_VERSION', 'unknown')
    })

def list_users(event, params):
    """ユーザー一覧"""
    users = db.query('USER#', limit=100)
    return response(200, {'users': users})

def create_user(event, params):
    """ユーザー作成"""
    body = parse_body(event)
    user_id = db.generate_id()

    item = {
        'PK': f'USER#{user_id}',
        'SK': 'PROFILE',
        'id': user_id,
        'name': body.get('name'),
        'email': body.get('email'),
    }
    db.put(item)

    return response(201, {'user': item})

def get_user(event, params):
    """ユーザー取得"""
    user_id = params.get('id')
    user = db.get(f'USER#{user_id}', 'PROFILE')

    if not user:
        return response(404, {'error': 'User not found'})

    return response(200, {'user': user})

def update_user(event, params):
    """ユーザー更新"""
    user_id = params.get('id')
    body = parse_body(event)

    updated = db.update(
        f'USER#{user_id}',
        'PROFILE',
        body
    )

    return response(200, {'user': updated})

def delete_user(event, params):
    """ユーザー削除"""
    user_id = params.get('id')
    db.delete(f'USER#{user_id}', 'PROFILE')

    return response(204, None)
```

### lib/db.py

```python
import boto3
import uuid
from typing import Any, Dict, List, Optional

class DynamoDB:
    def __init__(self, table_name: str):
        self.table = boto3.resource('dynamodb').Table(table_name)

    def generate_id(self) -> str:
        return str(uuid.uuid4())

    def get(self, pk: str, sk: str) -> Optional[Dict]:
        response = self.table.get_item(Key={'PK': pk, 'SK': sk})
        return response.get('Item')

    def put(self, item: Dict) -> None:
        self.table.put_item(Item=item)

    def update(self, pk: str, sk: str, updates: Dict) -> Dict:
        update_expr = 'SET ' + ', '.join(f'#{k} = :{k}' for k in updates.keys())
        expr_names = {f'#{k}': k for k in updates.keys()}
        expr_values = {f':{k}': v for k, v in updates.items()}

        response = self.table.update_item(
            Key={'PK': pk, 'SK': sk},
            UpdateExpression=update_expr,
            ExpressionAttributeNames=expr_names,
            ExpressionAttributeValues=expr_values,
            ReturnValues='ALL_NEW'
        )
        return response['Attributes']

    def delete(self, pk: str, sk: str) -> None:
        self.table.delete_item(Key={'PK': pk, 'SK': sk})

    def query(self, pk_prefix: str, limit: int = 100) -> List[Dict]:
        response = self.table.query(
            KeyConditionExpression='begins_with(PK, :pk)',
            ExpressionAttributeValues={':pk': pk_prefix},
            Limit=limit
        )
        return response.get('Items', [])
```

### lib/utils.py

```python
import json
from typing import Any, Dict, Optional

def response(status_code: int, body: Optional[Dict]) -> Dict:
    """API Gateway レスポンスを生成"""
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
        },
        'body': json.dumps(body) if body else ''
    }

def parse_body(event: Dict) -> Dict:
    """リクエストボディをパース"""
    body = event.get('body', '{}')
    if event.get('isBase64Encoded'):
        import base64
        body = base64.b64decode(body).decode('utf-8')
    return json.loads(body) if body else {}
```

---

# 第5部：UIを捨てる設計思想

## 「管理画面がない」ことの価値

### 従来の考え方

```
「管理画面を作らないと運用できない」
「UIがないとユーザーが使えない」
「ダッシュボードは必須」
```

### 発想の転換

```
管理画面を作る = 開発コスト + 保守コスト + セキュリティリスク

本当に必要なのは「データの操作」であって「UI」ではない
```

## UIがない方が顧客が喜ぶケース

### ケース1：バッチ連携システム

```
【UI あり】
顧客: 毎日手動で画面からデータをアップロード
    → 人的ミス、作業時間、担当者依存

【UI なし（API連携）】
顧客のシステム → API → 自動処理
    → 完全自動化、24時間稼働、ミスなし
```

**顧客の声：「毎日の作業がなくなって助かる」**

### ケース2：IoT/センサーデータ

```
【UI あり】
端末 → 管理画面 → 人が確認 → 対応

【UI なし（イベント駆動）】
端末 → API → Lambda → 異常検知 → 自動通知
    → 人間の介在なしで24時間監視
```

### ケース3：決済・課金システム

```
【UI あり】
管理者が画面から手動で請求処理
    → 月末に残業、ミスのリスク

【UI なし（自動化）】
月末 → EventBridge → Lambda → 請求処理 → メール送信
    → 完全自動、ミスなし、深夜でも動く
```

## API-Firstの設計

```
┌─────────────────────────────────────────────────────────────┐
│                        API (Lambda)                         │
│                                                             │
│   全ての操作は API 経由                                      │
│   UI は API の「1つの消費者」に過ぎない                       │
│                                                             │
└────────────────────────────┬────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │ Web UI  │         │ 外部連携 │         │  CLI   │
   │（必要なら）│         │ システム │         │ ツール  │
   └─────────┘         └─────────┘         └─────────┘
       ↑
   これは後から
   作ってもいい
```

## UIを作らない場合の代替手段

### 1. CLIツール

```python
#!/usr/bin/env python3
# cli.py - 管理用CLIツール

import click
import requests

API_BASE = "https://xxx.execute-api.ap-northeast-1.amazonaws.com/prod"

@click.group()
def cli():
    pass

@cli.command()
def list_users():
    """ユーザー一覧を表示"""
    response = requests.get(f"{API_BASE}/users")
    users = response.json()['users']
    for user in users:
        click.echo(f"{user['id']}: {user['name']} ({user['email']})")

@cli.command()
@click.argument('name')
@click.argument('email')
def create_user(name, email):
    """ユーザーを作成"""
    response = requests.post(f"{API_BASE}/users", json={
        'name': name,
        'email': email
    })
    user = response.json()['user']
    click.echo(f"Created: {user['id']}")

@cli.command()
@click.argument('user_id')
def delete_user(user_id):
    """ユーザーを削除"""
    response = requests.delete(f"{API_BASE}/users/{user_id}")
    click.echo(f"Deleted: {user_id}")

if __name__ == '__main__':
    cli()
```

```bash
# 使用例
./cli.py list-users
./cli.py create-user "田中太郎" "tanaka@example.com"
./cli.py delete-user abc123
```

### 2. Slackbot

```python
# Slack から操作できるようにする
import os
from slack_bolt import App
from slack_bolt.adapter.aws_lambda import SlackRequestHandler

app = App(
    token=os.environ["SLACK_BOT_TOKEN"],
    signing_secret=os.environ["SLACK_SIGNING_SECRET"]
)

@app.command("/user-list")
def list_users(ack, respond):
    ack()
    users = fetch_users()  # API呼び出し
    respond(f"ユーザー数: {len(users)}")

@app.command("/user-create")
def create_user(ack, respond, command):
    ack()
    name, email = command['text'].split()
    user = create_user_api(name, email)
    respond(f"作成しました: {user['id']}")

handler = SlackRequestHandler(app)

def lambda_handler(event, context):
    return handler.handle(event, context)
```

### 3. スプレッドシート連携

```python
# Google Sheets をUIとして使う
import gspread
from google.oauth2.service_account import Credentials

def sync_to_sheet():
    """DynamoDBのデータをスプレッドシートに同期"""
    gc = gspread.authorize(credentials)
    sheet = gc.open("ユーザー管理").sheet1

    users = fetch_all_users()

    # ヘッダー
    sheet.update('A1', [['ID', 'Name', 'Email', 'Created']])

    # データ
    rows = [[u['id'], u['name'], u['email'], u['created_at']] for u in users]
    sheet.update('A2', rows)
```

---

## コスト計算例

### 月間1万リクエストの場合

```
Lambda:
  リクエスト: 10,000回 × $0.0000002 = $0.002
  実行時間: 10,000回 × 100ms × 128MB = 128,000 GB-ms
           128,000 / 1024 / 1000 = 0.125 GB秒
           0.125 × $0.0000166667 = $0.000002
  合計: 約 $0.002（ほぼ無料枠内）

API Gateway (HTTP API):
  10,000回 × $1.00/100万 = $0.01

DynamoDB (オンデマンド):
  読み取り: 10,000回 × $0.25/100万 = $0.0025
  書き込み: 5,000回 × $1.25/100万 = $0.00625
  合計: 約 $0.01

月額合計: 約 $0.02（約3円）
```

### 月間100万リクエストの場合

```
Lambda:
  リクエスト: 無料枠内
  実行時間: 約 $1.67

API Gateway:
  100万 × $1.00/100万 = $1.00

DynamoDB:
  読み取り: 100万 × $0.25/100万 = $0.25
  書き込み: 50万 × $1.25/100万 = $0.625
  合計: 約 $0.88

月額合計: 約 $3.55（約500円）
```

---

## まとめ

### サーバーレスの利点

```
1. 激安（使った分だけ課金）
2. スケール自動（設定不要）
3. 運用不要（サーバー管理なし）
4. デプロイ簡単（コードをアップロードするだけ）
```

### Blue-Greenデプロイのポイント

```
1. バージョン: コードのスナップショット（不変）
2. エイリアス: バージョンへのポインタ
3. dev/prod: エイリアスで環境を分離
4. ロールバック: エイリアスを戻すだけ（秒速）
5. カナリア: トラフィックを徐々にシフト
```

### UIを捨てる判断基準

```
✅ 顧客が本当にUIを使うか？
✅ 自動化で置き換えられないか？
✅ CLI/Slack/スプレッドシートで十分では？
✅ UIの開発・保守コストは妥当か？
```

### 心がけ

```
1. 小さく始める（Lambda 1個から）
2. バージョンを必ず発行する
3. prod は直接変更しない
4. ロールバック手順を用意する
5. UIは最後の手段
```

---

## 参考リンク

- [AWS Lambda バージョンとエイリアス](https://docs.aws.amazon.com/lambda/latest/dg/configuration-aliases.html)
- [API Gateway ステージ変数](https://docs.aws.amazon.com/apigateway/latest/developerguide/stage-variables.html)
- [Lambda デプロイのベストプラクティス](https://docs.aws.amazon.com/lambda/latest/operatorguide/deployment.html)
- [Serverless Framework](https://www.serverless.com/)
- [AWS SAM](https://aws.amazon.com/serverless/sam/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
