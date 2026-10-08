# 基本設計書

## プロジェクト概要

| 項目 | 内容 |
|------|------|
| プロジェクト名 | Tomario（ホテル予約システム） |
| 対象環境 | nonprod（dev / staging / shared）／ prod（production） |
| アカウント構成 | nonprod・prodの2アカウント構成（判断の経緯は[ADR: アカウント戦略](../../tomario-steering/adr/infra/account/001-nonprod-prod-account-separation.md)参照） |
| リージョン | ap-northeast-1（東京） |
| アプリケーション | Python 3.12 / Flask 3.1（gunicornで起動するコンテナ） |
| フロントエンド | 静的ファイル（HTML/CSS/JavaScript）をS3に配置し、CloudFrontで配信 |
| データベース | MySQL 8.4（Amazon RDS） |
| IaC | Terraform |
| CI/CD | GitHub Actions + OIDC |

---

## 本書の位置付け

本書（basic-design配下）は、システムがどんな層・コンポーネントで構成され、どこに配置され、どう通信し、どう責務を分けるかという「構成と方針」を記述する。
実務と同じく本番運用（productionを一般公開して常時稼働させる状態）を前提とした本来の構成を記述する。ポートフォリオとして維持するために実際の運用が本書と異なる点は、[ポートフォリオとしての運用上の制約](../portfolio-constraints.md)に記載する。
以下は本書では扱わず、それぞれの文書へ分離している。

| 内容 | 記載先 |
|------|-------|
| なぜその構成・運用を選んだか（設計判断の理由） | ADR（`tomario-steering/adr/infra/`） |
| 環境ごとの具体的な設定値（リソース名・CPU/メモリ・インスタンスクラス・閾値・保持日数・コスト目安等） | [環境定義書](../environment-definitions/) |
| 運用手順・試験手順とその実績 | `tomario-steering/verification/`・`tomario-steering/release-management/` |

---

## 構成図

![アーキテクチャ図](https://github.com/Tomario-portfolio/tomario-infra/blob/main/docs/architecture/tomario-architecture.png?raw=true)

構成図の原本は`tomario-infra`リポジトリの`docs/architecture/`で管理している。

---

## 目次

| No | ドキュメント | 概要 |
|----|------------|------|
| 1 | [ネットワーク設計](network-high-level-spec.md) | VPC・サブネット・IGW・VPCエンドポイント・Flow Logs |
| 2 | [コンピューティング設計](compute-high-level-spec.md) | ECS Fargate・ECR・ALB・デプロイ方式 |
| 3 | [データベース設計](database-high-level-spec.md) | RDSの役割・接続方式・セキュリティ境界 |
| 4 | [セキュリティ設計](security-high-level-spec.md) | セキュリティグループ・IAM・機密情報管理・検出 |
| 5 | [可用性設計](availability-high-level-spec.md) | 可用性の基本方針・冗長構成 |
| 6 | [バックアップ・リストア設計](backup-high-level-spec.md) | バックアップ方針・RTO/RPO・リストア方式 |
| 7 | [モニタリング設計](monitoring-high-level-spec.md) | 監視方針・通知・ログ集約 |
| 8 | [命名規則](naming-high-level-spec.md) | リソース命名・タグ規則 |
| 9 | [コスト設計](cost-high-level-spec.md) | コスト最適化の方針・コスト監視 |
