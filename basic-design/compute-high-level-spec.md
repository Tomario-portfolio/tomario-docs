# コンピューティング

## 要件

## 基本方針

バックエンドアプリケーション（Flask）はECS Fargate上でコンテナとして動作させる。
ALBでトラフィックを受けてECSサービスに転送する。
コンテナイメージはECRで管理し、GitHub ActionsでビルドからデプロイまでをCI/CDで自動化する。
すべてのリソースはTerraformで管理し、手動操作による構成変更を行わない。

<!--
## EC2（旧）

アプリケーションサーバーとしてEC2を使用する。
インスタンスはASGで管理し、障害時の自動復旧を実現する。
SSM Session Managerを使用してEC2への接続を管理する。キーペアは使用しない。

dev環境ではコスト最適化のためパブリックサブネットに配置する。
prd環境の配置はTBD。

## Auto Scaling Group（ASG）（旧）

EC2インスタンスの可用性確保とコスト管理のためASGを使用する。
2AZにまたがってインスタンスを配置し、1AZ障害時もサービスを継続できる構成とする。

dev環境ではコスト最適化のためASGの最小台数を調整してインスタンスを停止できる運用とする。
-->

## ECR（Elastic Container Registry）

FlaskアプリケーションのコンテナイメージをECRで管理する。
GitHub ActionsでDockerイメージをビルドし、ECRにプッシュする。
ECSタスクはECRからイメージをプルして起動する。

イメージタグはIMMUTABLEとし、プッシュ時に脆弱性スキャンを自動実行する。
ライフサイクルポリシーにより最新5世代のみ保持し、古いイメージを自動削除する。

ECRは元々`modules/backend`内で管理していたが、`modules/ecr`として独立させ、`envs/nonprod/shared`に切り出した（2026-07-10、`terraform state mv`による無停止移行）。dev・stagingは同じ`tomario-app`リポジトリを共有する。ECRの`name`はAWS側でリネーム不可のため、既存名をそのまま維持している。
production環境は別アカウントのため、別リポジトリ`tomario-production-app`を新規作成する想定（クロスアカウント共有はしない）。staging→production昇格時は、stagingで検証済みのイメージをdigest指定でpull→re-tag→pushするpromoteジョブで対応し、再ビルドはしない。

### bootstrap_image

ECSタスク定義の初期イメージ（`var.bootstrap_image`）は、当初パブリックのECR Gallery（`public.ecr.aws/docker/library/nginx:latest`）を参照していたが、NAT Gatewayの無いVPC構成では到達できず、サービス再作成のたびにクラッシュループを起こす問題があった（2026-07-10発覚）。`tomario-app`（プライベートECR）内に`bootstrap`タグでプレースホルダーイメージを一度だけpushし、そちらを参照する形に修正した。

## ECS Fargate

ECS Fargateを使用してコンテナを実行する。
サーバー管理不要でアプリケーションの実行に集中できる。
ECSタスクはプライベートサブネットに配置し、インターネットからの直接アクセスを遮断する。
VPCエンドポイント経由でECR・CloudWatch Logs・Secrets Managerに接続する。

### ECS Cluster

ECSタスクをグループ管理するためのクラスターを作成する。

### ECS Task Definition

コンテナの実行仕様（イメージ・CPU・メモリ・ポート・環境変数）を定義する。
DB接続情報（ユーザー名・パスワード）およびFlask SECRET_KEYはSecrets Managerから取得し、コンテナの環境変数として注入する。

### ECS Service

タスク数の維持・ALBへの登録・ヘルスチェック管理をECSサービスで行う。
dev環境ではタスク数を1（`desired_count=1`）とする。コスト最適化のための意図的な設計であり、タスク障害時の再起動までの短時間ダウンタイムはdev環境では許容する。
staging環境ではタスク数を2（`desired_count=2`）とし、Application Auto Scaling（min=2/max=4、target CPU 70%）を有効化する。水平スケーリングの実挙動を検証することが目的で、CPU/メモリ自体はdevと同じ値（256/512）のまま変えない（垂直スケールはしない方針、2026-07-11決定）。
production環境はタスク数・Auto Scaling設定（min=2/max=4、target CPU 70%）をstagingからそのまま引き継ぐ。CPU/メモリのみstagingの2倍（512/1024）とする（詳細は[availability-high-level-spec.md](availability-high-level-spec.md)参照）。リリース前はstaging同様desired_countを0に落とすcost-stop運用とし、リリース時に常時稼働へ切り替える。
ALBのヘルスチェックにより異常なタスクへのルーティングを自動的に停止する。

### デプロイサーキットブレーカー（dev/staging共通、2026-07-11追加）

ECSサービスに`deployment_circuit_breaker { enable = true, rollback = true }`を設定し、デプロイ失敗時に自動的に直前の正常なリビジョンへロールバックする。Blue/Greenデプロイ（CodeDeploy）より導入コストが低く、その前段階の安全網として全環境共通で有効化する（AWS Well-Architectedベストプラクティスレビューで新規発見）。

### Blue/Greenデプロイ（Wave B、production初回構築時には導入しない）

デプロイ手順自体が変わる機能のため、production環境で初めて試すのはリスクが高いと判断し、staging環境で先に検証してからproductionへ展開する方針。ECSのデプロイコントローラーをCODE_DEPLOYに変更し、ALBに切替用のテストリスナー/ターゲットグループを追加する。

ただしproductionの初回構築（Wave A）では、既存のローリングアップデート＋デプロイサーキットブレーカーで安全網は確保できているため、Blue/Greenの導入は見送る。production稼働開始後の拡張（Wave B）としてstagingで検証し、production側へ展開する（`todo.md`のOPS-3/REL-5拡張を参照）。

## Application Load Balancer（ALB）

複数AZへのトラフィック分散と、ECSタスクの死活監視にALBを使用する。
ALBのヘルスチェックにより異常なタスクへのルーティングを自動的に停止する。

dev環境ではコスト削減のため作業時以外はALBを削除する運用とする。

## デプロイ設計

GitHub Actionsのワークフローでデプロイを自動化する。
tomario-appリポジトリへのpushをトリガーにDockerイメージをビルドし、ECRにプッシュする。
その後ECSサービスのローリングアップデートを実行してデプロイを完了する。

## ロールバック設計

デプロイ後に問題が発生した場合、旧イメージを指定したECSタスク定義の新リビジョンを登録し、ECSサービスを更新することで切り戻す。
ECRのIMMUTABLEタグにより旧イメージは最新5世代分保存されているため、いつでもロールバック可能である。

DBマイグレーションを含むデプロイをロールバックする場合はアプリとDBスキーマの整合性に注意が必要である。
具体的な手順はリリース前に作成する運用手順書に記載する。

<!--
## デプロイ設計（旧）

FlaskアプリケーションはEC2起動時のuser_dataによって自動的にセットアップする。
アプリケーションはポート8080で起動し、ALBのターゲットグループに登録する。
-->
