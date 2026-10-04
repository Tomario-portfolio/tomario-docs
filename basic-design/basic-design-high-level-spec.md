# 基本設計書

## プロジェクト概要

| 項目 | 内容 |
|------|------|
| プロジェクト名 | Tomario（ホテル予約システム） |
| 対象環境 | nonprod（dev / staging / shared）／ prod（production） |
| アカウント構成 | nonprod・prodの2アカウント構成。全環境構築済み。productionは一般公開前のため、dev/staging同様cost-stop/startで必要な時だけ起動する運用 |
| リージョン | ap-northeast-1（東京） |
| IaC | Terraform（required_version >= 1.10） |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |
| CI/CD | GitHub Actions + OIDC |

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
| 3 | [データベース設計](database-high-level-spec.md) | RDS・環境別構成・認証情報管理 |
| 4 | [セキュリティ設計](security-high-level-spec.md) | セキュリティグループ・IAM・機密情報管理・WAF等 |
| 5 | [可用性設計](availability-high-level-spec.md) | 冗長構成・稼働率目標 |
| 6 | [バックアップ・リストア設計](backup-high-level-spec.md) | RTO/RPO・バックアップ方針・リストア運用 |
| 7 | [モニタリング設計](monitoring-high-level-spec.md) | CloudWatchアラーム・SNS通知・ログ設計 |
| 8 | [命名規則](naming-high-level-spec.md) | リソース命名・タグ規則 |
| 9 | [コスト設計](cost-high-level-spec.md) | リソース別コスト方針・運用コスト試算 |

詳細なパラメータ（リソース名・ポート・設定値）は[環境定義書](../environment-definitions/)に記載する。
