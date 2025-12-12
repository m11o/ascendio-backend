# Design Document

---
**Purpose**: Provide sufficient detail to ensure implementation consistency across different implementers, preventing interpretation drift.
---

## Overview
本機能は、開発者が迅速かつ一貫性のあるローカル開発環境を構築できるようにするためのDocker Compose構成を提供します。
**Users**: バックエンド開発者は、この機能を使用してデータベース（PostgreSQL）をローカルで起動し、アプリケーションの開発やテストを行います。
**Impact**: 手動でのデータベースインストールや設定の手間を削減し、プロジェクト参加時のオンボーディング時間を短縮します。

### Goals
- プロジェクトルートでの `docker compose up` コマンド一発によるデータベース起動
- データの永続化による開発継続性の確保
- 環境変数による柔軟な接続設定

### Non-Goals
- 本番環境（Production）向けのDocker構成（セキュリティ要件やスケーリング設定などが異なるため）
- PostgreSQL以外のミドルウェア（Redisなど）の同時導入（今回はPostgreSQLのみスコープ）

## Architecture

### Architecture Pattern & Boundary Map
Docker Compose Specificationに基づき、単一の構成ファイルでサービスを定義します。

**Architecture Integration**:
- Selected pattern: Container-based Service Definition (Docker Compose)
- Domain/feature boundaries: インフラストラクチャ層としてアプリケーションコードから分離
- Steering compliance: 開発環境の標準化原則に準拠

### Technology Stack

| Layer | Choice / Version | Role in Feature | Notes |
|-------|------------------|-----------------|-------|
| Infrastructure / Runtime | Docker Compose | コンテナオーケストレーション | `compose.yml` 形式 |
| Data / Storage | PostgreSQL 17-alpine | リレーショナルデータベース | 軽量なAlpineイメージを採用 |

## Requirements Traceability

| Requirement | Summary | Components | Interfaces | Flows |
|-------------|---------|------------|------------|-------|
| 1.1 | Define `compose.yml` | `compose.yml` | Docker CLI | - |
| 1.2 | Schema version | `compose.yml` | - | - |
| 2.1 | Official Image | `db` service | - | - |
| 2.2 | Port Exposure | `db` service | TCP 5432 | - |
| 2.3 | Env Vars | `db` service | Env Vars | - |
| 2.4 | Volume Mount | `db` service | Volume | - |
| 2.5 | Container Name | `db` service | - | - |

## Components and Interfaces

### Infrastructure

#### Docker Compose Configuration (`compose.yml`)

| Field | Detail |
|-------|--------|
| Intent | Defines the database service and infrastructure configuration |
| Requirements | 1.1, 1.2, 2.1, 2.2, 2.3, 2.4, 2.5 |

**Responsibilities & Constraints**
- PostgreSQLコンテナの起動設定
- ホストマシンとのポートフォワーディング
- データの永続化ボリュームの管理

**Dependencies**
- Inbound: `docker compose` command — User triggers start/stop (Criticality: P0)
- Outbound: Docker Daemon — Executes containers (Criticality: P0)
- External: Docker Hub — Pulls `postgres` image (Criticality: P0)

**Contracts**: Service [ ] / API [ ] / Event [ ] / Batch [ ] / State [x]

##### State Management
- State model: PostgreSQL Data Directory (`/var/lib/postgresql/data`)
- Persistence: Docker Named Volume (`postgres_data`)
- Concurrency: Single instance (Development mode)

**Implementation Notes**
- Integration: プロジェクトルートに配置し、`.gitignore` に含めない（共有設定のため）。
- Validation: `docker compose config` コマンドで構文チェックが可能。
- Risks: ホストのポート5432が既に埋まっている場合、起動に失敗する。

##### Service Configuration (Detail)
```yaml
services:
  db:
    image: postgres:17-alpine
    container_name: ascendio-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
      POSTGRES_DB: ${DB_NAME:-ascendio}
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
```

## Data Models

### Physical Data Model
**For Relational Databases**:
- 本機能はデータベースサーバー自体を提供するものであり、特定のテーブルスキーマ定義はこの仕様のスコープ外とする。
- データはDocker Volume `postgres_data` 内に物理的に保存される。

## Error Handling

### Error Strategy
Docker Composeの標準エラー出力に依存する。

### Error Categories and Responses
**System Errors**:
- **Port Conflict**: `Bind for 0.0.0.0:5432 failed: port is already allocated` -> 既存のPostgreSQLプロセスを停止するか、`compose.yml` のポートを変更する。
- **Volume Permission**: ディレクトリ権限エラー -> Docker Desktopの設定またはボリュームの再作成を確認する。

## Testing Strategy

- Unit Tests: なし（設定ファイルのため）
- Integration Tests:
  - `docker compose up -d` を実行し、コンテナが `Up` 状態になることを確認する。
  - `docker compose down -v` でボリュームごと削除し、再作成できることを確認する。
  - `psql` コマンドまたはDBクライアントから `localhost:5432` に接続できることを確認する。
