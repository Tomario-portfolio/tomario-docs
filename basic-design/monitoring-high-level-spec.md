# モニタリング

## 基本方針

システムの異常をCloudWatchアラームで検知し、SNS経由で通知する。
インフラレベルの監視を行い、障害の早期発見と対応を可能にする。

## CloudWatchアラーム

以下のメトリクスを監視し、閾値を超えた場合にアラームを発報する。

| 監視対象 | メトリクス | 目的 | 対象環境 |
|---------|----------|------|---------|
| ALB | UnHealthyHostCount | ECSタスク障害の検知 | 全環境 |
| ECS | CPUUtilization | 高負荷状態の検知 | 全環境 |
| RDS | CPUUtilization | DB高負荷状態の検知 | 全環境 |
| WAF（CloudFront） | BlockedRequests | 攻撃・誤検知の急増の検知 | production（セキュリティスタック有効時のみ） |

アラーム発報時と復旧時の両方でSNSトピックに通知する。

## ダッシュボード

staging・productionでは、ECSの`RunningTaskCount`を表示するCloudWatchダッシュボードを作成し、Application Auto Scalingによるタスク数の増減を可視化する。
`RunningTaskCount`はContainer Insights有効時のみ配信されるメトリクスのため、全環境でContainer Insightsを有効化している（2026-09-29）。

## SNS

CloudWatchアラームの通知先としてSNSトピックを使用する。
アラーム発報時に管理者へメールで通知する。

## 性能分析

RDSのクエリ単位の性能分析にPerformance Insightsの導入を検討したが、`db.t3.micro`／`t3.small`／`t4g.micro`（MySQL 8.4.9）では未サポートのため見送っている（`t4g.medium`以上への恒久的な引き上げが必要でコスト方針に反するため、2026-09-29に導入を断念）。

## ログ設計

ECSタスクのアプリケーションログはCloudWatch Logsに収集する。複数行にわたるスタックトレースが分断されないよう、`awslogs-multiline-pattern`でログイベントの区切りを指定している。
ALBアクセスログ・VPC Flow Logs・CloudTrailは、ログ集約用のS3バケット（`modules/logging`）へ出力する。
保持期間は環境ごとに3段階とする（CloudWatch Logsの`retention_in_days`／S3ライフサイクルの`expiration.days`を変数化）。

| 環境 | 保持期間 | 判断理由 |
|------|---------|---------|
| dev | 7日 | 短期間で作り直す環境のため |
| staging | 30日 | 負荷テスト・障害試験の結果分析のため |
| production | 90日 | 気づくのが遅れたインシデントも追えるよう調査期間を確保 |

| 対象 | 出力先 |
|------|-------|
| ECS（Flaskアプリ） | CloudWatch Logs（`/ecs/tomario-{env}`） |
| ALB | S3（ログ集約バケット）にアクセスログを出力 |
| VPCネットワーク | S3（ログ集約バケット）にFlow Logsを直接出力 |
| CloudTrail | S3（ログ集約バケット）に出力（詳細は[security-high-level-spec.md](security-high-level-spec.md)参照） |
| WAF | CloudWatch Logs（`aws-waf-logs-tomario-production-*`、保持30日）。productionでセキュリティスタック有効時のみ |
