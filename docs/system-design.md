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

### Web Framework: Gin

```
github.com/gin-gonic/gin
```

Goで最も広く使われているWebフレームワーク。高速なルーティング、ミドルウェアサポート、JSONバインディングを提供。

### ORM: GORM

```
gorm.io/gorm
gorm.io/driver/postgres
```

GoのデファクトスタンダードORM。マイグレーション、リレーション、トランザクション管理を提供。`driver/postgres`はPostgreSQL用ドライバ。

### JWT: golang-jwt

```
github.com/golang-jwt/jwt/v5
```

JWT（JSON Web Token）の生成・検証ライブラリ。RS256（公開鍵暗号）、HS256（共通鍵）両方に対応。v5は最新のメジャーバージョン。

### Password Hash: bcrypt

```
golang.org/x/crypto/bcrypt
```

Go公式の拡張パッケージ。パスワードのハッシュ化に業界標準のbcryptアルゴリズムを使用。ソルト生成・検証を内蔵。

### UUID: google/uuid

```
github.com/google/uuid
```

Google製のUUID生成ライブラリ。RFC 4122準拠のUUID v4（ランダム）生成に使用。主キーの生成に利用。

### Validation: validator

```
github.com/go-playground/validator/v10
```

構造体のバリデーションライブラリ。Ginに組み込まれており、タグベースでメール形式、必須チェック等を定義可能。

### Config: Viper

```
github.com/spf13/viper
```

設定管理ライブラリ。環境変数、YAMLファイル、JSONファイルなど複数のソースから設定を読み込める。12-Factor App対応。

### Redis: go-redis

```
github.com/redis/go-redis/v9
```

Redis公式クライアント。Refresh Token管理、レートリミット、セッション管理に使用。コネクションプーリング対応。

### 依存関係まとめ

| カテゴリ | ライブラリ | 用途 |
|---------|-----------|------|
| Web | gin | HTTPサーバー、ルーティング |
| ORM | gorm | DB操作、マイグレーション |
| 認証 | golang-jwt/jwt | アクセストークン発行・検証 |
| セキュリティ | x/crypto/bcrypt | パスワードハッシュ化 |
| ID生成 | google/uuid | 主キー生成 |
| バリデーション | validator | リクエスト検証 |
| 設定 | viper | 環境変数・設定ファイル読み込み |
| キャッシュ | go-redis | Refresh Token、レートリミット |

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

### ローカル開発環境（DevContainer）

**方針：**

- VSCode DevContainerで開発環境を統一
- PostgreSQL、RedisもDevContainer内で起動
- Go開発ツール（Air, delve等）がプリインストール済み
- 外部サービスに依存せず、完全にローカルで完結

**構成図：**

```
┌─────────────────────────────────────────────────────────┐
│                   DevContainer                          │
│                                                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  Go開発コンテナ (auth_platform)                  │    │
│  │  ・Go 1.23.x                                    │    │
│  │  ・Air (hot reload)                             │    │
│  │  ・delve (debugger)                             │    │
│  │  localhost:8080                                 │    │
│  └────────────────────────────────────────────────┘    │
│           │                        │                    │
│           ▼                        ▼                    │
│  ┌──────────────┐         ┌──────────────┐             │
│  │  PostgreSQL  │         │    Redis     │             │
│  │  :5432       │         │    :6379     │             │
│  └──────────────┘         └──────────────┘             │
└─────────────────────────────────────────────────────────┘
```

**DevContainer構成：**

```
.devcontainer/
├── .env                  # 環境変数
├── .extensions/          # VSCode拡張キャッシュ
│   └── .gitignore
├── compose.yml           # Go開発コンテナ + PostgreSQL + Redis
├── devcontainer.json     # VSCode設定
└── go/
    └── Dockerfile        # Go開発環境
```

**開発ツール：**

| ツール | 用途 |
|--------|------|
| Air | Goのホットリロード |
| delve | デバッガ |
| golangci-lint | リンター |
| psql | PostgreSQLクライアント（コンテナ内蔵） |

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

## アーキテクチャ

### レイヤードアーキテクチャ + CQRS

本プロジェクトではレイヤードアーキテクチャを採用し、Service層でCQRS（Command Query Responsibility Segregation）パターンを適用する。

```
┌─────────────────────────────────────────────────────────────┐
│  Handler層（API境界）                                        │
│  dto/ : Request / Response の型定義                         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Service層（ビジネスロジック）                                │
│  ├── command/ : 更新系（Create, Update, Delete）            │
│  └── query/   : 参照系（Get, List）                         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Repository層（データアクセス）                               │
│  entity/ : DBエンティティ（テーブルと1:1対応）                │
└─────────────────────────────────────────────────────────────┘
```

### 各レイヤーの責務

| レイヤー | ディレクトリ | 責務 |
|---------|-------------|------|
| Handler | `handler/` | HTTPリクエスト/レスポンスの処理、バリデーション |
| DTO | `handler/dto/` | API境界の型定義（Request/Response） |
| Command | `service/command/` | 更新系ビジネスロジック（副作用あり） |
| Query | `service/query/` | 参照系ビジネスロジック（副作用なし） |
| Repository | `repository/` | データベースアクセス |
| Entity | `repository/entity/` | DBテーブルに対応する構造体 |

### CQRS分離の理由

| 観点 | Command（更新系） | Query（参照系） |
|------|------------------|----------------|
| トランザクション | 必要 | 不要なことが多い |
| キャッシュ | 難しい | しやすい |
| スケーリング | 書き込みDB | 読み取りレプリカ可 |
| 処理の複雑さ | バリデーション、整合性確保 | シンプル |

### 型の変換フロー

各レイヤーは独自の型を持ち、レイヤー間で明示的に変換する。

```
LoginRequest (dto)
      │
      ▼ Handler で変換
service.LoginInput
      │
      ▼ Command で処理
entity.User (repository から取得)
      │
      ▼ Handler で変換
LoginResponse (dto)
```

## リポジトリ構成

```
auth-platform/
├── app/                          # Goアプリケーション
│   ├── cmd/
│   │   └── server/
│   │       └── main.go           # エントリーポイント
│   ├── internal/
│   │   ├── config/               # 設定
│   │   │   └── config.go
│   │   ├── handler/              # HTTPハンドラ
│   │   │   ├── auth.go
│   │   │   ├── user.go
│   │   │   ├── tenant.go
│   │   │   ├── router.go         # ルーティング定義
│   │   │   └── dto/              # Request/Response
│   │   │       ├── auth.go
│   │   │       ├── user.go
│   │   │       └── tenant.go
│   │   ├── service/              # ビジネスロジック
│   │   │   ├── command/          # 更新系
│   │   │   │   ├── auth.go       # Login, Logout, RefreshToken
│   │   │   │   ├── user.go       # CreateUser, UpdateUser, DeleteUser
│   │   │   │   └── tenant.go     # CreateTenant, UpdateTenant
│   │   │   └── query/            # 参照系
│   │   │       ├── user.go       # GetUser, ListUsers
│   │   │       └── tenant.go     # GetTenant, ListTenants
│   │   ├── repository/           # DB操作
│   │   │   ├── user.go
│   │   │   ├── tenant.go
│   │   │   ├── role.go
│   │   │   ├── refresh_token.go
│   │   │   └── entity/           # DBエンティティ
│   │   │       ├── user.go
│   │   │       ├── tenant.go
│   │   │       ├── role.go
│   │   │       └── refresh_token.go
│   │   └── middleware/           # ミドルウェア
│   │       ├── auth.go           # JWT検証
│   │       ├── logging.go
│   │       └── ratelimit.go
│   ├── pkg/                      # 外部公開可能なパッケージ
│   │   └── jwt/                  # JWT発行・検証（他サービスでも使える）
│   │       └── jwt.go
│   ├── db/
│   │   └── migrations/           # DBマイグレーション
│   ├── api/
│   │   └── openapi.yaml          # API定義
│   ├── .air.toml                 # ホットリロード設定
│   ├── go.mod
│   ├── go.sum
│   └── Dockerfile
├── infra/                        # IaC（後日追加）
│   └── terraform/
│       ├── environments/
│       │   ├── dev/
│       │   └── prod/
│       └── modules/
├── docs/                         # ドキュメント
│   ├── design-principles.md
│   └── system-design.md
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

### 前提条件

- Docker / Docker Compose
- VSCode + Dev Containers拡張機能

### セットアップ手順

```bash
# 1. リポジトリのクローン
git clone https://github.com/mori-hisayuki/auth-platform.git
cd auth-platform

# 2. VSCodeで開く
code .

# 3. DevContainerで開く
#    VSCodeの通知 "Reopen in Container" をクリック
#    または コマンドパレット → "Dev Containers: Reopen in Container"

# 4. コンテナ内でサーバー起動
cd app
air
# または
go run cmd/server/main.go
```

### 接続情報（DevContainer内）

| サービス | 接続先 |
|---------|-------|
| PostgreSQL | postgres:5432 (user: auth, password: password, db: auth) |
| Redis | redis:6379 |
| API | http://localhost:8080 |

### 環境変数（自動設定済み）

```
DATABASE_URL=postgres://auth:password@postgres:5432/auth?sslmode=disable
REDIS_URL=redis://redis:6379
```
