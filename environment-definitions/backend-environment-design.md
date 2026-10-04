# backend 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

ALB・ターゲットグループ・ECS・ECRをまとめて管理するコンポーネント。
コスト管理のため、作業時以外はALB削除・ECSタスク停止・VPCエンドポイント削除を行う運用とする。

---

## セキュリティグループ

### ALB-SG

| 項目 | dev | staging | production |
|------|-----|---------|---|
| SG名 | tomario-dev-alb-sg | tomario-staging-alb-sg | tomario-production-alb-sg |

| 方向 | プロトコル | ポート | 送信元 |
|------|----------|-------|--------|
| インバウンド | TCP | 80 | 0.0.0.0/0 |
| アウトバウンド | 全て | 全て | 0.0.0.0/0 |

### ECS-SG

| 項目 | dev | staging | production |
|------|-----|---------|---|
| SG名 | tomario-dev-ecs-sg | tomario-staging-ecs-sg | tomario-production-ecs-sg |

| 方向 | プロトコル | ポート | 送信元 |
|------|----------|-------|--------|
| インバウンド | TCP | 8080 | ALB-SG |
| アウトバウンド | 全て | 全て | 0.0.0.0/0 |

---

## ALB

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ALB名 | tomario-dev-alb | tomario-staging-alb | tomario-production-alb |
| スキーム | internet-facing | internet-facing | internet-facing |
| 配置サブネット | パブリックサブネット×2 | パブリックサブネット×2 | パブリックサブネット×2 |
| アクセスログ | 有効。S3（ログ集約バケット、`logging-environment-design.md`参照）の`/alb/`プレフィックスへ出力（2026-07-13実装） | 同左 | 同左 |
| 削除保護（`enable_deletion_protection`） | 無効（固定値、変数化していない） | 無効（同左） | 無効（同左、一般公開前のため） |
| 運用 | 未使用時は削除（~$17/月） | 未使用時は削除（負荷テスト実施時のみ起動） | 一般公開前：未使用時は削除（dev/staging同様のcost-stop対象）。公開後：常時起動（~$18/月、デモ可能な状態を維持） |

**一般公開時の切り替え方針：** `enable_deletion_protection`はコード上`false`固定（変数化していない）。公開して常時稼働へ切り替えるタイミングで、`modules/backend/alb.tf`を直接`true`に変更する想定（詳細は[cost-high-level-spec.md](../basic-design/cost-high-level-spec.md)参照）。

### リスナー

| プロトコル | ポート | アクション |
|----------|-------|---------|
| HTTP | 80 | ターゲットグループへ転送 |

### ターゲットグループ

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ターゲットグループ名 | tomario-dev-tg | tomario-staging-tg | tomario-production-tg |
| ターゲットタイプ | ip | ip | ip |
| プロトコル | HTTP | HTTP | HTTP |
| ポート | 8080 | 8080 | 8080 |
| ヘルスチェックパス | /health | /health | /health |
| ヘルスチェック間隔 | 30秒 | 30秒 | 30秒 |
| 正常判定しきい値 | 2回 | 2回 | 2回 |
| 異常判定しきい値 | 2回 | 2回 | 2回 |

---

## ECR

2026-07-10より`modules/ecr`として独立し、`envs/nonprod/shared`へ移設済み。詳細は[`ecr-environment-design.md`](ecr-environment-design.md)を参照。

---

## ECS Cluster

| 項目 | dev | staging | production |
|------|-----|---------|---|
| クラスター名 | tomario-dev-cluster | tomario-staging-cluster | tomario-production-cluster |
| Container Insights | 有効 | 有効 | 有効 |

全環境共通モジュール（`modules/backend/ecs.tf`）で無条件に有効化している（2026-09-29、PR #89）。`RunningTaskCount`等のメトリクスがContainer Insights有効時のみ配信される仕様のため、ダッシュボードでのタスク数可視化に必要。

---

## ECS Task Definition

| 項目 | dev | staging | production |
|------|-----|---------|---|
| タスク定義名 | tomario-dev-task | tomario-staging-task | tomario-production-task |
| 起動タイプ | FARGATE | FARGATE | FARGATE |
| CPU | 256（0.25 vCPU） | 256（変更なし。垂直スケールはしない方針、2026-07-11決定） | 256（変更なし。stagingと同一） |
| メモリ | 512 MB | 512 MB（変更なし） | 512 MB（stagingと同一） |
| CPUアーキテクチャ | ARM64（Graviton） | ARM64（Graviton） | ARM64（Graviton） |
| bootstrap_image | `tomario-app`（プライベートECR）の`bootstrap`タグ | 同左（旧：public ECR Galleryを参照しNAT無しVPCでpull失敗していたため修正、2026-07-10） | `tomario-production-app`（別リポジトリ）の`bootstrap`タグ |
| コンテナポート | 8080 | 8080 | 8080 |
| ログドライバー | awslogs（CloudWatch Logs、保持7日） | awslogs（CloudWatch Logs、保持30日） | awslogs（CloudWatch Logs、保持90日） |

機密情報はSecrets Managerから取得し、非機密の接続情報は環境変数として直接渡す。

| 環境変数 | 取得元 | 備考 |
|---------|--------|------|
| DB_HOST | 環境変数（Terraform） | RDSエンドポイント（非機密） |
| DB_PORT | 環境変数（ハードコード） | `3306`（固定値） |
| DB_NAME | 環境変数（ハードコード） | `tomario`（固定値） |
| DB_USER | Secrets Manager | RDSマスターユーザー名（`manage_master_user_password`でAWSが管理。自動スケジュールでの定期ローテーションは未設定、CLIで手動トリガー可能） |
| DB_PASSWORD | Secrets Manager | RDSマスターパスワード（同上） |
| SECRET_KEY | Secrets Manager | Flask セッション署名キー（**手動登録、ローテーション未設定**。SEC-8として改善を検討中） |

---

## ECS Service

| 項目 | dev | staging | production |
|------|-----|---------|---|
| サービス名 | tomario-dev-service | tomario-staging-service | tomario-production-service |
| 起動タイプ | FARGATE | FARGATE | FARGATE |
| タスク数（desired） | 1（作業時） / 0（未使用時） | 2（作業時） / 0（未使用時） | 2（作業時）/ 0（未使用時）。stagingと同じcost-stop対象（リリース後は常時2で固定） |
| Application Auto Scaling | 無効 | 有効（min=2, max=4, target CPU 70%） | 有効（min=2, max=4, target CPU 70%、stagingと同一設定） |
| 配置サブネット | プライベートサブネット×2 | プライベートサブネット×2 | プライベートサブネット×2 |
| パブリックIP割り当て | 無効 | 無効 | 無効 |
| デプロイ方式 | ローリングアップデート | ローリングアップデート | ローリングアップデート（初回構築時はBlue/Greenを導入しない。Wave BでstagingへCodeDeployを先行導入・検証してからproductionへ展開） |
| デプロイサーキットブレーカー | 有効（`enable=true, rollback=true`） | 有効（同左） | 有効（同左） |

デプロイサーキットブレーカーは2026-07-11にベストプラクティスレビューで新規発見し、dev/staging共通で追加することにした。デプロイ失敗時に自動ロールバックする安全網で、Blue/Greenデプロイより導入コストが低い。

---

## IAM

### Task Execution Role（ECSサービスが使用）

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ロール名 | tomario-dev-task-exec-role | tomario-staging-task-exec-role | tomario-production-task-exec-role |

| 付与ポリシー | 用途 |
|------------|------|
| AmazonECSTaskExecutionRolePolicy | ECRからのイメージ取得・CloudWatch Logsへの書き込み |
| secretsmanager:GetSecretValue | DB接続情報の取得 |

### Task Role（アプリケーションが使用）

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ロール名 | tomario-dev-task-role | tomario-staging-task-role | tomario-production-task-role |

現時点では最小権限。アプリケーションがAWSサービスを直接呼び出す場合に追加する。

---

## TBD解消事項

| 項目 | 内容 |
|------|------|
| staging/production CPU/メモリ | devと同一（256/512）に決定。垂直スケールはせず、水平（タスク数・Auto Scaling）のみで負荷対応する方針（2026-07-11） |
