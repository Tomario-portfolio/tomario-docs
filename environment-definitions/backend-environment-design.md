# backend 環境定義

ALB・ターゲットグループ・ECS・関連IAMロールを管理するコンポーネント（`modules/backend`）。ECRは[ecr-environment-design.md](ecr-environment-design.md)で定義する。

ALB・ECSタスクはcost-stopの対象（未使用時はALB削除・ECSタスク数0）。productionも一般公開前は同じ扱いとする（[cost-high-level-spec.md](../basic-design/cost-high-level-spec.md)参照）。

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

送信元はCloudFrontのIPに限定せず、リスナールールの`X-Origin-Verify`ヘッダー検証でCloudFront経由以外を拒否する（下記リスナー参照）。

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
| アクセスログ | S3ログ集約バケットの`/alb/`プレフィックス（[logging-environment-design.md](logging-environment-design.md)参照） | 同左 | 同左 |
| 削除保護（`enable_deletion_protection`） | 無効（`modules/backend/alb.tf`で`false`固定） | 無効 | 無効（一般公開時に`true`へ変更する） |

### リスナー

| プロトコル | ポート | デフォルトアクション |
|----------|-------|------------------|
| HTTP | 80 | 固定レスポンス 403（`Forbidden`） |

### リスナールール

| 優先度 | 条件 | アクション |
|-------|------|----------|
| 1 | HTTPヘッダー`X-Origin-Verify`がCloudFrontと共有するシークレット値と一致 | ターゲットグループへ転送 |

シークレット値は環境ごとに`random_password`で生成し、CloudFrontのカスタムヘッダーとALBのリスナールールに同じ値を設定する。

### ターゲットグループ

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ターゲットグループ名 | tomario-dev-tg | tomario-staging-tg | tomario-production-tg |
| ターゲットタイプ | ip | ip | ip |
| プロトコル / ポート | HTTP / 8080 | HTTP / 8080 | HTTP / 8080 |
| ヘルスチェックパス | /health | /health | /health |
| ヘルスチェック間隔 | 30秒 | 30秒 | 30秒 |
| 正常判定しきい値 | 2回 | 2回 | 2回 |
| 異常判定しきい値 | 2回 | 2回 | 2回 |

---

## ECS Cluster

| 項目 | dev | staging | production |
|------|-----|---------|---|
| クラスター名 | tomario-dev-cluster | tomario-staging-cluster | tomario-production-cluster |
| Container Insights | 有効 | 有効 | 有効 |

---

## ECS Task Definition

| 項目 | dev | staging | production |
|------|-----|---------|---|
| タスク定義名 | tomario-dev-task | tomario-staging-task | tomario-production-task |
| 起動タイプ | FARGATE | FARGATE | FARGATE |
| CPU | 256（0.25 vCPU） | 256 | 256 |
| メモリ | 512 MB | 512 MB | 512 MB |
| CPUアーキテクチャ | ARM64（Graviton） | ARM64（Graviton） | ARM64（Graviton） |
| bootstrap_image | `tomario-app`（プライベートECR）の`bootstrap`タグ | 同左 | `tomario-production-app`の`bootstrap`タグ |
| コンテナポート | 8080 | 8080 | 8080 |
| ログドライバー | awslogs（`/ecs/tomario-dev`） | awslogs（`/ecs/tomario-staging`） | awslogs（`/ecs/tomario-production`） |

ログの保持期間・マルチライン設定は[monitoring-environment-design.md](monitoring-environment-design.md)を参照。

### 環境変数

| 環境変数 | 取得元 | 備考 |
|---------|--------|------|
| DB_HOST | 環境変数（Terraform） | RDSエンドポイント |
| DB_PORT | 環境変数（固定値） | `3306` |
| DB_NAME | 環境変数（固定値） | `tomario` |
| DB_USER | Secrets Manager | RDSマスターユーザー名（`manage_master_user_password`でAWSが管理） |
| DB_PASSWORD | Secrets Manager | RDSマスターパスワード（同上） |
| SECRET_KEY | Secrets Manager | Flaskセッション署名キー。全環境で初期値を`random_password`で生成し、以降は90日ごとに自動ローテーション（下記「SECRET_KEYの自動ローテーション」参照） |
| SECRET_KEY_PREVIOUS | Secrets Manager（`AWSPREVIOUS`） | 1つ前の鍵。アプリは`SECRET_KEY_FALLBACKS`に入れ、ローテーション前に発行されたcookieも受け付ける |

---

## ECS Service

| 項目 | dev | staging | production |
|------|-----|---------|---|
| サービス名 | tomario-dev-service | tomario-staging-service | tomario-production-service |
| 起動タイプ | FARGATE | FARGATE | FARGATE |
| タスク数（desired） | 1（作業時） / 0（未使用時） | 2（作業時） / 0（未使用時） | 2（作業時） / 0（未使用時）。公開後は常時2 |
| Application Auto Scaling | 無効 | 有効（min=2, max=4, target CPU 70%） | 有効（min=2, max=4, target CPU 70%） |
| 配置サブネット | プライベートサブネット×2 | プライベートサブネット×2 | プライベートサブネット×2 |
| パブリックIP割り当て | 無効 | 無効 | 無効 |
| デプロイ方式 | ローリングアップデート | ローリングアップデート | ローリングアップデート |
| デプロイサーキットブレーカー | 有効（`enable=true, rollback=true`） | 有効（同左） | 有効（同左） |

デプロイ方式の判断は[ADR: デプロイの安全網](../../tomario-steering/adr/infra/backend/002-deployment-safety-net.md)を参照。

---

## IAM

### Task Execution Role（ECSサービスが使用）

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ロール名 | tomario-dev-task-exec-role | tomario-staging-task-exec-role | tomario-production-task-exec-role |

| 付与ポリシー | 用途 |
|------------|------|
| AmazonECSTaskExecutionRolePolicy | ECRからのイメージ取得・CloudWatch Logsへの書き込み |
| secretsmanager:GetSecretValue | DB接続情報・SECRET_KEYの取得（対象シークレットのARNに限定） |

### Task Role（アプリケーションが使用）

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ロール名 | tomario-dev-task-role | tomario-staging-task-role | tomario-production-task-role |

| 付与権限 | 用途 |
|---------|------|
| ssmmessages:CreateControlChannel / CreateDataChannel / OpenControlChannel / OpenDataChannel | ECS Exec（障害対応・DBメンテナンス時のコンテナ接続） |

---

## SECRET_KEYの自動ローテーション（`modules/secret-rotation`）

| 項目 | dev | staging | production |
|------|-----|---------|---|
| ローテーション間隔 | 90日 | 90日 | 90日 |
| ローテーションLambda | tomario-dev-flask-secret-rotation | tomario-staging-flask-secret-rotation | tomario-production-flask-secret-rotation |
| 鍵の生成 | `GetRandomPassword`（50文字） | 同左 | 同左 |
| ローテーション後の反映 | ECSサービスが稼働中ならforce-new-deploymentでタスクを入れ替える（cost-stop中は次のcost-startで反映） | 同左 | 同左 |
| 1つ前の鍵 | `AWSPREVIOUS`を環境変数`SECRET_KEY_PREVIOUS`として渡し、アプリが`SECRET_KEY_FALLBACKS`に設定（ローテーション後もログインが切れない） | 同左 | 同左 |

---

## 未解決事項

なし
