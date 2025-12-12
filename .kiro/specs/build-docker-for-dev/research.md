# Research & Design Decisions

---
**Purpose**: Capture discovery findings, architectural investigations, and rationale that inform the technical design.
---

## Summary
- **Feature**: build-docker-for-dev
- **Discovery Scope**: Simple Addition / Extension
- **Key Findings**:
  - PostgreSQLの最新安定版は17系/18系だが、開発環境の安定性と互換性を考慮し、Docker Hubで広く利用可能な最新バージョン（例: `postgres:16` または `postgres:17`）をベースにする。
  - Docker Composeの仕様（Compose Specification）ではトップレベルの `version` 指定は非推奨化されつつある。
  - 必須環境変数は `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`。
  - データ永続化パスは `/var/lib/postgresql/data`。

## Research Log

### Docker & PostgreSQL Version
- **Context**: 開発環境に適したバージョンと定義ファイルの形式を決定するため。
- **Sources Consulted**: Docker Hub Official Image, Docker Compose Specification
- **Findings**:
  - **PostgreSQL**: 公式イメージの環境変数 (`POSTGRES_PASSWORD` 必須) を確認。Alpine版 (`-alpine`) は軽量で開発用に適している。
  - **Compose**: 最新の `compose.yml` では `version: "3.8"` などの記述はオプショナル、または推奨されない（Compose Specification準拠）。
- **Implications**:
  - `compose.yml` を使用（`docker-compose.yml` よりも推奨される傾向）。
  - トップレベル `version` は省略する。
  - 軽量化のため `postgres:16-alpine` または `17-alpine` を採用（特段の制約がなければ最新安定版）。

## Architecture Pattern Evaluation

| Option | Description | Strengths | Risks / Limitations | Notes |
|--------|-------------|-----------|---------------------|-------|
| Standard Compose | プロジェクトルートに単一の `compose.yml` を配置 | シンプル、標準的 | 環境ごとの差異管理が必要な場合に複雑化 | 今回は開発環境専用のため最適 |

## Design Decisions

### Decision: PostgreSQL Image Selection
- **Context**: 使用するDockerイメージの選定。
- **Alternatives Considered**:
  1. `postgres:latest` - 常に最新だが、メジャーバージョンアップで互換性が壊れるリスクあり。
  2. `postgres:17-alpine` - 軽量、バージョン固定で安定。
  3. `bitnami/postgresql` - 非ルートユーザー実行などが考慮されているが設定が独自。
- **Selected Approach**: `postgres:17-alpine` (またはその時点の安定版Alpine)
- **Rationale**: 開発環境の起動速度（ダウンロードサイズ）と環境の一貫性を重視。
- **Trade-offs**: Alpine Linux特有のライブラリ差異が稀にあるが、DB単体利用ではほぼ問題なし。

### Decision: Data Persistence Strategy
- **Context**: コンテナ再起動時のデータ保持。
- **Alternatives Considered**:
  1. ホストディレクトリマウント (`./pgdata:/var/lib/postgresql/data`) - データが直接見えるが、権限問題が起きやすい。
  2. 名前付きボリューム (`postgres_data:/var/lib/postgresql/data`) - Docker管理下で安全、権限トラブルが少ない。
- **Selected Approach**: 名前付きボリューム
- **Rationale**: ホストOS（特にMac/Windows）またぐ際のファイルシステムパフォーマンスと権限管理の煩雑さを回避するため。

## Risks & Mitigations
- **Port Conflict**: ホストの5432が既に使用されている場合、起動に失敗する。
  - **Mitigation**: `compose.yml` のポートマッピングを環境変数で上書き可能にするか、ドキュメントで停止を案内する。今回は標準設定 (`5432:5432`) とし、競合時はユーザー対処とする。
