# モニタリング

## 要件

## 基本方針

システムの異常をCloudWatchアラームで検知し、SNS経由で通知する。
インフラレベルの監視を行い、障害の早期発見と対応を可能にする。

## CloudWatchアラーム

以下のメトリクスを監視し、閾値を超えた場合にアラームを発報する。

| 監視対象 | メトリクス | 目的 |
|---------|----------|------|
| ECS | CPU使用率 | 高負荷状態の検知 |
| ALB | 異常ホスト数 | ECSタスク障害の検知 |
| RDS | CPU使用率 | DB高負荷状態の検知 |
| ECS（staging） | RunningTaskCount | Application Auto Scalingの挙動可視化（負荷テスト中にタスク数が増減した証跡を残す） |

## SNS

CloudWatchアラームの通知先としてSNSトピックを使用する。
アラーム発報時に管理者へ通知する。

## ログ設計

ECSタスクのアプリケーションログはCloudWatch Logsに収集する。
ALBアクセスログ・VPC Flow Logs・CloudTrailは、ログ集約用に新設したS3バケット（`modules/logging`）へ出力する（2026-07-13、ログ管理の棚卸しで実装）。
保持期間は環境ごとに3段階とする（`retention_in_days`／S3ライフサイクルの`expiration.days`を変数化）。

| 環境 | 保持期間 | 判断理由 |
|------|---------|---------|
| dev | 7日 | 短期間で作り直す環境のため。なお元々明確な根拠のない初期設定値だった |
| staging | 30日 | 負荷テスト・障害試験の結果分析のため延長 |
| production | 90日 | 気づくのが遅れたインシデントも追えるよう調査期間を確保 |

| 対象 | 対応 |
|------|------|
| ECS（Flaskアプリ） | CloudWatch Logs連携済み（/ecs/tomario-{env}） |
| ALB | 実装済み。アクセスログをS3（ログ集約バケット）へ出力 |
| VPCネットワーク | 実装済み。Flow LogsをS3（ログ集約バケット）へ直接出力（元はCloudWatch Logsだったが2026-07-13にS3直接出力へ変更） |
| CloudTrail | 実装済み。同じログ集約バケットへ出力（詳細はsecurity参照） |
| RDS | production環境のみ、エラーログをCloudWatch Logsへエクスポートする（`enabled_cloudwatch_logs_exports = ["error"]`）。追加コストは軽微なため導入する。dev・stagingは見送り |
