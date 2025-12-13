# Auth Platform

マイクロサービスアーキテクチャにおける認証・認可の中央基盤

## 概要

全サービスに共通する認証機能を提供する独立した認証基盤。ユーザー、テナント、共通ロールを一元管理し、JWTベースの認証を提供する。

## 技術スタック

| カテゴリ | 技術 |
|---------|------|
| 言語 | Go |
| フレームワーク | Gin |
| ORM | GORM |
| データベース | PostgreSQL |
| キャッシュ | Redis |
| API | REST |
| 本番インフラ | GCP (Cloud Run) |

## リポジトリ構成

```
auth-platform/
├── app/                  # Goアプリケーション
├── infra/                # IaC（Terraform）
├── docs/                 # ドキュメント
├── docker-compose.yaml
└── README.md
```

## ドキュメント

- [設計思想](docs/design-principles.md) - 認証基盤の設計方針と原則
- [システム設計](docs/system-design.md) - 技術スタック、アーキテクチャ、開発フェーズ

## 開発環境のセットアップ

### 前提条件

- Go 1.22+
- Docker / Docker Compose
- Air（ホットリロード用、オプション）

### セットアップ手順

```bash
# 1. リポジトリのクローン
git clone https://github.com/mori-hisayuki/auth-platform.git
cd auth-platform

# 2. Docker Composeで依存サービスを起動
docker-compose up -d

# 3. 依存関係のインストール
cd app
go mod download

# 4. サーバー起動
go run cmd/server/main.go

# または Air でホットリロード
air
```

### 接続情報（ローカル）

| サービス | 接続先 |
|---------|-------|
| PostgreSQL | localhost:5432 (user: auth, password: password, db: auth) |
| Redis | localhost:6379 |
| API | http://localhost:8080 |

## ライセンス

Private

