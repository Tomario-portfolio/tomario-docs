# monitoring 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

SNS・CloudWatchアラームを管理するコンポーネント。
インフラレベルの異常を検知し、管理者へ通知する。

---

## SNS

| 項目 | dev | staging | production |
|------|-----|---------|---|
| トピック名 | tomario-dev-alarm | tomario-staging-alarm | tomario-production-alarm |
| 通知先 | 管理者メールアドレス | 管理者メールアドレス | 管理者メールアドレス |

---

## CloudWatchアラーム

| アラーム名 | 監視対象 | メトリクス | 閾値 | 評価期間 | 通知先 |
|-----------|---------|----------|------|---------|-------|
| tomario-{env}-alb-unhealthy-host | ALB | UnHealthyHostCount | ≥ 1 | 1分×1回 | SNSトピック |
| tomario-{env}-ecs-cpu | ECS（サービス） | CPUUtilization | ≥ 80% | 5分×2回 | SNSトピック |
| tomario-{env}-rds-cpu | RDS | CPUUtilization | ≥ 80% | 5分×2回 | SNSトピック |
| tomario-staging-ecs-running-tasks | ECS（サービス） | RunningTaskCount | ― | ― | Application Auto Scalingの挙動可視化用（負荷テスト中にタスク数が増減した証跡を残す）。**既知の問題**：Container Insightsが無効なため実際には表示されない（`todo.md` #1、staging単独では未修正） |
| tomario-production-ecs-running-tasks | ECS（サービス） | RunningTaskCount | ― | ― | productionはCluster作成時からContainer Insightsを有効化するため（`backend-environment-design.md`参照）、staging同様の問題は発生しない想定 |

アラーム発報時（`alarm_actions`）と復旧時（`ok_actions`）の両方でSNSトピックに通知する。

---

## ログ設計

| 対象 | 状況 | ロググループ／出力先 | 保持期間 |
|------|------|------------|---------|
| ECS（Flaskアプリ） | 収集済み | /ecs/tomario-{env}（CloudWatch Logs） | dev=7日／staging=30日／production=90日（予定） |
| VPCネットワーク | 収集済み（Flow Logs） | S3ログ集約バケットの`/flow-logs/`（`logging-environment-design.md`参照） | dev=7日／staging=30日／production=90日（予定） |
| ALB | 収集済み（2026-07-13実装） | S3ログ集約バケットの`/alb/` | dev=7日／staging=30日／production=90日（予定） |
| RDS | dev/staging：未収集（見送り）。production：収集する（決定） | production限定でCloudWatch Logsへエラーログをエクスポート（`enabled_cloudwatch_logs_exports = ["error"]`） | production=90日（他ログと統一） |

`retention_in_days`を変数化し、環境ごとに3段階の保持期間を設定する（2026-07-11決定。元は7日固定で明確な根拠のない初期設定だった）。

---

## TBD解消事項

なし
