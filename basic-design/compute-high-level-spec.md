# コンピューティング

## 基本方針

バックエンドアプリケーション（Flask）はECS Fargate上でコンテナとして動作させる。
ALBでトラフィックを受けてECSサービスに転送する。
コンテナイメージはECRで管理し、GitHub ActionsでビルドからデプロイまでをCI/CDで自動化する。
すべてのリソースはTerraformで管理し、手動操作による構成変更を行わない。

コンピューティング基盤の選定理由（EC2 + Auto Scaling Groupからの移行を含む）は[ADR: コンピューティング基盤選択](../../tomario-steering/adr/infra/backend/001-compute-platform-selection.md)を参照。

## ECR（Elastic Container Registry）

FlaskアプリケーションのコンテナイメージをECRで管理する。
GitHub ActionsでDockerイメージをビルドし、ECRにプッシュする。
ECSタスクはECRからイメージをプルして起動する。

イメージタグはIMMUTABLEとし、プッシュ時に脆弱性スキャンを自動実行する。
ライフサイクルポリシーで保持世代数を制限し、古いイメージを自動削除する。

リポジトリはアカウント単位で持ち、nonprod（dev/staging）は1つのリポジトリを共有、productionは別アカウントのため別リポジトリとする（クロスアカウント共有はしない）。リポジトリ名・管理場所は[環境定義書](../environment-definitions/ecr-environment-design.md)を参照。

ECSタスク定義の初期イメージ（`bootstrap_image`）は、各環境のプライベートECR内のプレースホルダーイメージを参照する（経緯は[ADR: bootstrap_imageのプライベートECR参照化](../../tomario-steering/adr/infra/network/002-bootstrap-image-private-ecr.md)参照）。

## ECS Fargate

ECSタスクはプライベートサブネットに配置し、インターネットからの直接アクセスを遮断する。
VPCエンドポイント経由でECR・CloudWatch Logs・Secrets Managerに接続する。

### ECS Cluster

ECSタスクをグループ管理するためのクラスターを環境ごとに作成する。
Container Insightsを有効化し、タスク数等のメトリクスでオートスケーリングの挙動を可視化する。

### ECS Task Definition

コンテナの実行仕様（イメージ・CPU・メモリ・ポート・環境変数）を定義する。
CPUアーキテクチャはARM64（Graviton）とする。
DB接続情報およびFlask SECRET_KEYはSecrets Managerから取得し、コンテナの環境変数として注入する。

### ECS Service

タスク数の維持・ALBへの登録・ヘルスチェック管理をECSサービスで行う。

| 環境 | 冗長化 | Auto Scaling |
|------|-------|-------------|
| dev | 単一タスク（短時間のダウンタイムを許容） | 無効 |
| staging | 複数タスクを2AZに分散 | 有効（CPU使用率のターゲット追跡） |
| production | stagingと同一構成 | 有効（stagingと同一設定） |

負荷への対応は水平スケール（タスク数）のみで行い、タスクサイズの垂直スケールはしない。
タスク数・CPU/メモリ・スケーリング閾値の具体値は[環境定義書](../environment-definitions/backend-environment-design.md)を参照。

## Application Load Balancer（ALB）

複数AZへのトラフィック分散と、ECSタスクの死活監視にALBを使用する。
ALBのヘルスチェックにより異常なタスクへのルーティングを自動的に停止する。
CloudFrontからのリクエストのみを受け付けるため、CloudFrontが付与する`X-Origin-Verify`ヘッダーをリスナールールで検証する。

ALBは「停止」ができないため、未使用時は削除する運用の対象とする（[cost-high-level-spec.md](cost-high-level-spec.md)参照）。

## デプロイ設計

GitHub Actions（`tomario-app`リポジトリの`deploy.yml`）でデプロイを自動化する。

| 環境 | デプロイ方法 |
|------|------------|
| dev / staging | mainへのpushをトリガーにDockerイメージをビルドしてnonprodのECRにpushし、ECSサービスのローリングアップデートで反映 |
| production | 再ビルドはしない。staging検証済みのイメージをdigest指定で昇格（re-tagしてproductionのECRへpush）し、ECSサービスを更新する |

デプロイ失敗時の安全網として、全環境でECSのデプロイサーキットブレーカー（自動ロールバック付き）を有効化する。Blue/Greenデプロイ（CodeDeploy）は導入しない。判断の経緯は[ADR: デプロイの安全網](../../tomario-steering/adr/infra/backend/002-deployment-safety-net.md)を参照。

## ロールバック設計

デプロイ後に問題が発生した場合、旧イメージを指定したECSタスク定義の新リビジョンを登録し、ECSサービスを更新することで切り戻す。
ECRのIMMUTABLEタグにより旧イメージが保持されているため、保持世代の範囲内でいつでもロールバック可能である。

DBマイグレーションを含むデプロイをロールバックする場合はアプリとDBスキーマの整合性に注意が必要である。
