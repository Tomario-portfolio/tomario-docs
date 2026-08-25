# 基本設計書

## プロジェクト概要

| 項目 | 内容 |
|------|------|
| プロジェクト名 | Tomario（ホテル予約システム） |
| 対象環境 | nonprod（dev / staging / shared）／ prod（production、未着手） |
| アカウント構成 | nonprod・prodの2アカウント構成。nonprodアカウント内のディレクトリ・state分離、prodアカウント作成・bootstrap（OIDC・IAMロール）は完了済み。production環境（`envs/prod/production`）のリソース構築のみ未着手 |
| リージョン | ap-northeast-1（東京） |
| IaC | Terraform（required_version ~> 1.5） |
| AWSプロバイダー | hashicorp/aws ~> 6.0（最新：6.50.0） |
| CI/CD | GitHub Actions + OIDC |

---

## 構成図

![アーキテクチャ図](../../reference/architecture.drawio.png)

---

## 目次

| No | ドキュメント | 概要 |
|----|------------|------|
| 1 | [ネットワーク設計](network.md) | VPC・サブネット・IGW・ルートテーブル |
| 2 | [コンピューティング設計](compute.md) | ECS Fargate・ECR・ALB・デプロイ方式 |
| 3 | [データベース設計](database.md) | RDS・バックアップ・認証情報管理 |
| 4 | [セキュリティ設計](security.md) | セキュリティグループ・IAM・機密情報管理 |
| 5 | [可用性設計](availability.md) | 冗長構成・稼働率目標 |
| 6 | [バックアップ・リストア設計](backup.md) | RTO/RPO・バックアップ方針 |
| 7 | [モニタリング設計](monitoring.md) | CloudWatchアラーム・SNS通知・ログ設計 |
| 8 | [命名規則](naming.md) | リソース命名・タグ規則 |
| 9 | [コスト設計](cost.md) | リソース別コスト方針・運用コスト試算 |
