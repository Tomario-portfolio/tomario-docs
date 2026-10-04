# コンピューティング

## 基本方針

バックエンドアプリケーション（Flask）はECS Fargate上でコンテナとして動作させる。
ALBでトラフィックを受けてECSサービスに転送する。
コンテナイメージはECRで管理し、GitHub ActionsでビルドからデプロイまでをCI/CDで自動化する。
すべてのリソースはTerraformで管理し、手動操作による構成変更を行わない。

当初はEC2 + Auto Scaling Groupで構築したが、OSパッチ管理の不要化・SSHポート廃止による攻撃面の削減・CI/CDとの親和性を理由にECS Fargateへ移行した。

## ECR（Elastic Container Registry）

FlaskアプリケーションのコンテナイメージをECRで管理する。
GitHub ActionsでDockerイメージをビルドし、ECRにプッシュする。
ECSタスクはECRからイメージをプルして起動する。

イメージタグはIMMUTABLEとし、プッシュ時に脆弱性スキャンを自動実行する。
ライフサイクルポリシーにより最新5世代のみ保持し、古いイメージを自動削除する。

| 環境 | リポジトリ | 管理場所 |
|------|----------|---------|
| dev / staging | `tomario-app`（共有） | `envs/nonprod/shared` |
| production | `tomario-production-app` | `envs/prod/production/ecr` |

productionは別アカウントのため別リポジトリとし、クロスアカウント共有はしない。
ECRの`name`はAWS側でリネーム不可のため、nonprod側は既存名`tomario-app`をそのまま維持している。

### bootstrap_image

ECSタスク定義の初期イメージ（`var.bootstrap_image`）は、各環境のプライベートECR内に`bootstrap`タグで置いたプレースホルダーイメージを参照する。
当初はパブリックのECR Gallery（`public.ecr.aws/docker/library/nginx:latest`）を参照していたが、NAT Gatewayの無いVPC構成では到達できず、サービス再作成のたびにクラッシュループを起こしたため変更した（2026-07-10）。

## ECS Fargate

ECS Fargateを使用してコンテナを実行する。
サーバー管理不要でアプリケーションの実行に集中できる。
ECSタスクはプライベートサブネットに配置し、インターネットからの直接アクセスを遮断する。
VPCエンドポイント経由でECR・CloudWatch Logs・Secrets Managerに接続する。

### ECS Cluster

ECSタスクをグループ管理するためのクラスターを作成する。
全環境でContainer Insightsを有効化し、`RunningTaskCount`等のメトリクスでオートスケーリングの挙動を可視化する。

### ECS Task Definition

コンテナの実行仕様（イメージ・CPU・メモリ・ポート・環境変数）を定義する。
CPUアーキテクチャはARM64（Graviton）とする。
DB接続情報（ユーザー名・パスワード）およびFlask SECRET_KEYはSecrets Managerから取得し、コンテナの環境変数として注入する。

### ECS Service

タスク数の維持・ALBへの登録・ヘルスチェック管理をECSサービスで行う。

| 環境 | タスク数 | Auto Scaling | CPU / メモリ |
|------|--------|-------------|-------------|
| dev | 1 | 無効 | 256 / 512 |
| staging | 2 | 有効（min=2/max=4、target CPU 70%） | 256 / 512 |
| production | 2 | 有効（min=2/max=4、target CPU 70%） | 256 / 512 |

- dev：コスト最適化のための意図的な設計であり、タスク障害時の再起動までの短時間ダウンタイムは許容する
- staging：水平スケーリングの実挙動を検証することが目的。垂直スケールはせず、水平（タスク数）のみで負荷に対応する方針（2026-07-11決定）
- production：stagingで検証済みの値をそのまま引き継ぐ。一般公開前はcost-stopでタスク数0に落とし、必要な時だけ起動する

ALBのヘルスチェックにより異常なタスクへのルーティングを自動的に停止する。

### デプロイサーキットブレーカー

ECSサービスに`deployment_circuit_breaker { enable = true, rollback = true }`を設定し、デプロイ失敗時に自動的に直前の正常なリビジョンへロールバックする（全環境共通）。Blue/Greenデプロイ（CodeDeploy）より導入コストが低く、その前段階の安全網として位置付ける。

### Blue/Greenデプロイ（将来対応）

デプロイ手順自体が変わる機能のため、production環境で初めて試すのはリスクが高いと判断し、導入する場合はstaging環境で先に検証してからproductionへ展開する方針とする。
現時点ではローリングアップデート＋デプロイサーキットブレーカーで安全網を確保できているため、導入は見送っている。

## Application Load Balancer（ALB）

複数AZへのトラフィック分散と、ECSタスクの死活監視にALBを使用する。
ALBのヘルスチェックにより異常なタスクへのルーティングを自動的に停止する。
CloudFrontからのリクエストのみを受け付けるため、CloudFrontが付与する`X-Origin-Verify`ヘッダーをリスナールールで検証する。

ALBは「停止」ができないため、全環境で作業時以外は削除する運用とする（cost-stop対象）。そのため削除保護は全環境で無効としている。

## デプロイ設計

GitHub Actions（`tomario-app`リポジトリの`deploy.yml`）でデプロイを自動化する。

| 環境 | デプロイ方法 |
|------|------------|
| dev / staging | mainへのpushをトリガーにDockerイメージをビルドし、`tomario-app`にpush。ECSサービスのローリングアップデートで反映 |
| production | 再ビルドはしない。promoteジョブでstaging検証済みのイメージをdigest指定でpull→re-tag→`tomario-production-app`へpushし、ECSサービスを更新する。productionのECSサービスが停止中（cost-stop中）の場合はデプロイをスキップする |

## ロールバック設計

デプロイ後に問題が発生した場合、旧イメージを指定したECSタスク定義の新リビジョンを登録し、ECSサービスを更新することで切り戻す。
ECRのIMMUTABLEタグにより旧イメージは最新5世代分保存されているため、いつでもロールバック可能である。

DBマイグレーションを含むデプロイをロールバックする場合はアプリとDBスキーマの整合性に注意が必要である。
