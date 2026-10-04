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
| tomario-{env}-ecs-running-tasks | ECS（サービス） | RunningTaskCount | ― | ― | Application Auto Scalingの挙動可視化用（負荷テスト中にタスク数が増減した証跡を残す）。全環境でContainer Insightsを有効化済み（2026-09-29、PR #89）のため表示される（詳細は[backend-environment-design.md](backend-environment-design.md)参照） |

アラーム発報時（`alarm_actions`）と復旧時（`ok_actions`）の両方でSNSトピックに通知する。

---

## ログ設計

| 対象 | 状況 | ロググループ／出力先 | 保持期間 |
|------|------|------------|---------|
| ECS（Flaskアプリ） | 収集済み。マルチライン例外（スタックトレース）は`awslogs-multiline-pattern`で1イベントにまとめる（2026-09-29、PR #90） | /ecs/tomario-{env}（CloudWatch Logs） | dev=7日／staging=30日／production=90日 |
| VPCネットワーク | 収集済み（Flow Logs） | S3ログ集約バケットの`/flow-logs/`（`logging-environment-design.md`参照） | dev=7日／staging=30日／production=90日 |
| ALB | 収集済み（2026-07-13実装） | S3ログ集約バケットの`/alb/` | dev=7日／staging=30日／production=90日 |
| RDS | 未収集（見送り） | ― | ― |

`retention_in_days`を変数化し、環境ごとに3段階の保持期間を設定する（2026-07-11決定。元は7日固定で明確な根拠のない初期設定だった）。

---

## TBD解消事項

なし
