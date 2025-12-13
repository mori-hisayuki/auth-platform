# 認証基盤 システム設計

## 技術スタック

### 言語: Go

**選定理由：**

- 認証基盤は「一度作ったらあまり変更しない」性質を持ち、開発速度より安定性・パフォーマンスを重視
- シングルバイナリでデプロイが単純、Dockerイメージも軽量
- 認証系ライブラリ（golang-jwt, bcrypt等）のエコシステムが充実
- 長期運用・保守に向いている
- 採用市場での人材確保がしやすい

**比較検討した言語：**

| 言語 | 評価 | 理由 |
|------|------|------|
| Go | ◎ 採用 | バランスが良い、実績豊富 |
| Rust | △ | パフォーマンス最高だが学習コスト・エコシステムに難 |
| TypeScript | △ | CPU負荷の高い処理（bcrypt等）との相性が悪い |

### フレームワーク: Gin

**選定理由：**

- Goで最もメジャーなWebフレームワーク（GitHub 80k+ stars）
- 情報量が豊富で、トラブルシューティングが容易
- 軽量かつ高速

### ORM: GORM

**選定理由：**

- Goで最もメジャーなORM
- 認証基盤のテーブル構成は比較的シンプルで、ORMで十分対応可能
- マイグレーション機能を内蔵

### データベース: PostgreSQL

**選定理由：**

- 安定性と実績
- UUID型のネイティブサポート
- JSON型によるスキーマレスデータの柔軟な保存

### キャッシュ: Redis

**用途：**

- Refresh Tokenの管理
- レートリミット
- セッション情報（必要に応じて）

### API: REST

**選定理由：**

- ブラウザから直接アクセスする必要があり、gRPCは不適
- 認証基盤のAPIはシンプルで、gRPCの恩恵（型安全、ストリーミング等）が薄い
- 将来的にサービス間通信でgRPCが必要になれば、内部向けエンドポイントとして追加可能

## ライブラリ

```
Web Framework:  github.com/gin-gonic/gin
ORM:            gorm.io/gorm
PostgreSQL:     gorm.io/driver/postgres
JWT:            github.com/golang-jwt/jwt/v5
Password Hash:  golang.org/x/crypto/bcrypt
UUID:           github.com/google/uuid
Validation:     github.com/go-playground/validator/v10
Config:         github.com/spf13/viper
Redis:          github.com/redis/go-redis/v9
```

## インフラ構成

### 開発方針

まずローカル環境（Docker Compose）で動作するものを構築し、本番環境（GCP）は後から整備する。

```
開発フロー:
1. Docker Composeでローカル開発 ← 今ここから
2. 機能が安定したらGCPへデプロイ
3. 本番運用開始
```

---

### ローカル開発環境（Docker Compose）

**方針：**

- PostgreSQLとRedisをDockerで起動
- Goアプリはホストで直接実行（ホットリロード対応）
- 外部サービスに依存せず、完全にローカルで完結

**構成図：**

```
┌─────────────────────────────────────────────────────────┐
│                   Local Machine                         │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Go App (host)                                  │    │
│  │  go run / air (hot reload)                      │    │
│  │  localhost:8080                                 │    │
│  └────────────────────────────────────────────────┘    │
│           │                        │                    │
│           ▼                        ▼                    │
│  ┌─────────────────────────────────────────────┐       │
│  │           Docker Compose                     │       │
│  │  ┌──────────────┐    ┌──────────────┐       │       │
│  │  │  PostgreSQL  │    │    Redis     │       │       │
│  │  │  :5432       │    │    :6379     │       │       │
│  │  └──────────────┘    └──────────────┘       │       │
│  └─────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

**docker-compose.yaml：**

```yaml
services:
  postgres:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: auth
      POSTGRES_USER: auth
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

**開発ツール：**

| ツール | 用途 |
|--------|------|
| Air | Goのホットリロード |
| psql / pgAdmin | DBクライアント |
| redis-cli | Redisクライアント |

---

### 本番環境（GCP）※後日構築

**選定理由：**

- GoはGoogle製言語であり、GCPとの親和性が高い
- Cloud RunはGoの軽量バイナリ・高速起動と相性が良い
- ECSと比較して設定がシンプルで運用負荷が低い
- 従量課金でコスト効率が良い

**GCP構成図：**

```
┌─────────────────────────────────────────────────────────┐
│                        GCP                              │
│                                                         │
│  ┌───────────────┐     ┌────────────────────────────┐  │
│  │ Cloud Load    │────▶│  Cloud Run                  │  │
│  │ Balancing     │     │  ┌──────────┐ ┌──────────┐ │  │
│  └───────────────┘     │  │Instance 1│ │Instance 2│ │  │
│                        │  └──────────┘ └──────────┘ │  │
│                        │  (オートスケール)            │  │
│                        └────────────────────────────┘  │
│                               │            │           │
│                               ▼            ▼           │
│                        ┌──────────┐  ┌──────────┐     │
│                        │Cloud SQL │  │Memorystore│    │
│                        │PostgreSQL│  │  (Redis)  │    │
│                        │  HA構成  │  └──────────┘     │
│                        └──────────┘                    │
└─────────────────────────────────────────────────────────┘
```

**構成要素：**

| コンポーネント | サービス | 備考 |
|---------------|---------|------|
| ロードバランサ | Cloud Load Balancing | HTTPS終端、Cloud Runと自動統合 |
| コンピューティング | Cloud Run | コンテナベース、オートスケーリング |
| データベース | Cloud SQL PostgreSQL | HA構成で高可用性 |
| キャッシュ | Memorystore (Redis) | Refresh Token管理、レートリミット |
| シークレット管理 | Secret Manager | DB接続情報、JWT秘密鍵等 |

**Cloud Runの設定方針：**

```yaml
min_instances: 1          # 常時1台は起動（コールドスタート回避）
max_instances: 10         # 負荷に応じてスケール
cpu: 1                    # 1 vCPU
memory: 512Mi             # 認証処理には十分
timeout: 60s
concurrency: 80           # 1インスタンスあたりの同時リクエスト数
```

## リポジトリ構成

```
auth-platform/
├── app/                      # Goアプリケーション
│   ├── cmd/
│   │   └── server/
│   │       └── main.go
│   ├── internal/
│   │   ├── handler/          # HTTPハンドラ
│   │   │   ├── auth.go
│   │   │   ├── user.go
│   │   │   └── tenant.go
│   │   ├── service/          # ビジネスロジック
│   │   │   ├── auth.go
│   │   │   ├── user.go
│   │   │   └── tenant.go
│   │   ├── repository/       # DB操作
│   │   │   ├── user.go
│   │   │   ├── tenant.go
│   │   │   └── refresh_token.go
│   │   ├── model/            # ドメインモデル
│   │   │   ├── user.go
│   │   │   ├── tenant.go
│   │   │   └── role.go
│   │   ├── middleware/       # ミドルウェア
│   │   │   ├── auth.go
│   │   │   ├── logging.go
│   │   │   └── ratelimit.go
│   │   └── config/
│   │       └── config.go
│   ├── pkg/
│   │   └── jwt/              # JWT発行・検証（他サービスでも使える）
│   │       └── jwt.go
│   ├── db/
│   │   └── migrations/
│   ├── api/
│   │   └── openapi.yaml
│   ├── go.mod
│   └── Dockerfile
├── infra/                    # IaC（後日追加）
│   └── terraform/
│       ├── environments/
│       │   ├── dev/
│       │   └── prod/
│       └── modules/
├── docs/                     # ドキュメント
│   ├── design-principles.md
│   └── system-design.md
├── docker-compose.yaml
└── README.md
```

## データベーススキーマ（概要）

```
┌─────────────┐       ┌─────────────┐
│   tenants   │       │    roles    │
├─────────────┤       ├─────────────┤
│ id (PK)     │       │ id (PK)     │
│ name        │       │ name        │
│ plan        │       │ description │
│ created_at  │       └─────────────┘
│ updated_at  │              │
└─────────────┘              │
       │                     │
       │ 1:N                 │
       ▼                     │
┌─────────────┐              │
│    users    │              │
├─────────────┤              │
│ id (PK)     │              │
│ tenant_id   │──────────────┤
│ email       │              │
│ password    │       ┌──────┴──────┐
│ name        │       │ user_roles  │
│ created_at  │       ├─────────────┤
│ updated_at  │       │ user_id     │
└─────────────┘       │ role_id     │
       │              └─────────────┘
       │ 1:N
       ▼
┌────────────────┐
│ refresh_tokens │
├────────────────┤
│ id (PK)        │
│ user_id        │
│ token          │
│ expires_at     │
│ created_at     │
└────────────────┘
```

## 開発フェーズ

### Phase 1: 認証の基本（MVP）

- [ ] プロジェクトセットアップ（Go modules, Docker, etc.）
- [ ] テナント作成 API
- [ ] ユーザー登録 API（テナント紐付け）
- [ ] ログイン → JWT発行
- [ ] トークンリフレッシュ
- [ ] ログアウト（Refresh Token無効化）
- [ ] JWT検証ミドルウェア

### Phase 2: 管理機能

- [ ] ユーザーCRUD API
- [ ] テナントCRUD API
- [ ] ロール管理 API
- [ ] ユーザーへのロール割り当て
- [ ] パスワードリセット

### Phase 3: 拡張

- [ ] OAuth連携（Google, GitHub等）
- [ ] MFA（多要素認証）
- [ ] 監査ログ
- [ ] 管理画面

### Phase 4: サービス間連携（必要に応じて）

- [ ] gRPCエンドポイント追加
- [ ] ユーザー情報取得 API（内部向け）

## 非機能要件

### 可用性（本番環境）

- Cloud Runのオートスケーリングで冗長化
- Cloud SQL HA構成（リージョン内の自動フェイルオーバー）
- JWTの署名検証は各サービスで完結（認証基盤停止時も既存トークンで一定時間動作）
- min_instances: 1 でコールドスタートを回避

### セキュリティ

- パスワードはbcryptでハッシュ化
- JWTは公開鍵暗号方式（RS256）で署名
- HTTPS必須（本番環境：Cloud Load Balancingで終端）
- レートリミット実装
- Secret Managerで機密情報を管理（本番環境）

### 監視（本番環境）

- Cloud Runの組み込みヘルスチェック
- Cloud Loggingでログ集約
- Cloud Monitoringでメトリクス監視
- Error Reportingでエラー追跡

## ローカル開発の始め方

```bash
# 1. リポジトリのクローン
git clone https://github.com/mori-hisayuki/auth-platform.git
cd auth-platform

# 2. Docker Composeで依存サービスを起動
docker-compose up -d

# 3. 依存関係のインストール
cd app
go mod download

# 4. サーバー起動（ホットリロード）
air
# または
go run cmd/server/main.go
```
