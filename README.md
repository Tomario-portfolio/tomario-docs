# tomario-docs

Tomario（ホテル予約システム）の設計ドキュメントをまとめたリポジトリ。
要件定義から基本設計、環境ごとの具体的な設定値までを扱う。

---

## フォルダ構成

| フォルダ・ファイル | 内容 |
|------------------|------|
| [requirements/](requirements/) | 要件定義。システムが「何を満たすべきか」 |
| [basic-design/](basic-design/) | 基本設計。システムを「どんな構成・方針で作るか」 |
| [environment-definitions/](environment-definitions/) | 環境定義。dev/staging/productionの各環境に「実際にどんな値で作るか」 |
| [portfolio-constraints.md](portfolio-constraints.md) | ポートフォリオとして維持するために、設計書と実際の運用が異なる点と、本番運用へ切り替える方法 |

### requirements/

| ファイル | 内容 |
|---------|------|
| [requirements.md](requirements/requirements.md) | 要件定義書。スコープ・ユーザーストーリー・機能要件・画面一覧・API一覧・DB設計・将来拡張（バックログ） |
| [non-functional-requirements.md](requirements/non-functional-requirements.md) | 非機能要件定義書。可用性・性能・運用保守性・セキュリティ・移行性・環境・コストの7カテゴリ |

### basic-design/

[basic-design-high-level-spec.md](basic-design/basic-design-high-level-spec.md)が全体の概要と目次。

| ファイル | 内容 |
|---------|------|
| [network-high-level-spec.md](basic-design/network-high-level-spec.md) | ネットワーク（VPC・サブネット・VPCエンドポイント・Flow Logs） |
| [compute-high-level-spec.md](basic-design/compute-high-level-spec.md) | コンピューティング（ECS Fargate・ECR・ALB・デプロイ方式） |
| [database-high-level-spec.md](basic-design/database-high-level-spec.md) | データベース（RDSの役割・接続方式・セキュリティ境界） |
| [security-high-level-spec.md](basic-design/security-high-level-spec.md) | セキュリティ（セキュリティグループ・暗号化・IAM・機密情報管理・検出） |
| [availability-high-level-spec.md](basic-design/availability-high-level-spec.md) | 可用性（冗長構成・稼働率の目標） |
| [backup-high-level-spec.md](basic-design/backup-high-level-spec.md) | バックアップ・リストア（RPO/RTO・リストア方式） |
| [monitoring-high-level-spec.md](basic-design/monitoring-high-level-spec.md) | モニタリング（監視対象・通知・ログ集約） |
| [naming-high-level-spec.md](basic-design/naming-high-level-spec.md) | 命名規則・タグ規則 |
| [cost-high-level-spec.md](basic-design/cost-high-level-spec.md) | コスト（停止・削除の方針・コスト監視） |

### environment-definitions/

[README.md](environment-definitions/README.md)が索引（共通事項・環境境界）。
コンポーネントごとに、`network`・`backend`・`database`・`frontend`・`ecr`・`logging`・`monitoring`・`security`・`cost`の環境定義書がある。

---

## 文書の役割分担

| 知りたいこと | 記載先 |
|------------|-------|
| 何を満たすべきか | 要件定義書（requirements/） |
| どんな構成・方針で作るか | 基本設計書（basic-design/） |
| 各環境に実際にどんな値で作るか | 環境定義書（environment-definitions/） |
| なぜその構成・運用にしたか | ADR（[tomario-steering](https://github.com/Tomario-portfolio/tomario-steering)の`adr/`） |
| 試験・運用の手順と実績 | [tomario-steering](https://github.com/Tomario-portfolio/tomario-steering)の`verification/`・`release-management/` |

---

## 関連リポジトリ

| リポジトリ | 内容 |
|----------|------|
| [tomario-infra](https://github.com/Tomario-portfolio/tomario-infra) | インフラのTerraformコードとCI/CD |
| [tomario-app](https://github.com/Tomario-portfolio/tomario-app) | アプリケーション（Flask）とフロントエンド、デプロイ用のCI/CD |
| [tomario-steering](https://github.com/Tomario-portfolio/tomario-steering) | ADR・試験・リリース管理 |
