# monitoring 環境定義

SNS・CloudWatchアラーム・CloudWatchダッシュボードを管理するコンポーネント（`modules/monitoring`）と、ログの保持設定を定義する。

---

## SNS

| 項目 | dev | staging | production |
|------|-----|---------|---|
| トピック名 | tomario-dev-alarm | tomario-staging-alarm | tomario-production-alarm |
| WAFアラーム用トピック（us-east-1） | ― | ― | tomario-production-waf-alarm（セキュリティスタック有効時のみ） |
| 通知先 | 管理者メールアドレス | 管理者メールアドレス | 管理者メールアドレス |

---

## CloudWatchアラーム

| アラーム名 | 監視対象 | メトリクス | 閾値 | 評価期間 | 対象環境 |
|-----------|---------|----------|------|---------|---------|
| tomario-{env}-alb-unhealthy-host | ALB | UnHealthyHostCount | ≥ 1 | 1分×1回 | 全環境 |
| tomario-{env}-ecs-cpu | ECS（サービス） | CPUUtilization | ≥ 80% | 5分×2回 | 全環境 |
| tomario-{env}-rds-cpu | RDS | CPUUtilization | ≥ 80% | 5分×2回 | 全環境 |
| tomario-{env}-waf-blocked | WAF（CloudFront用Web ACL） | BlockedRequests（Sum） | > 10 | 5分×1回 | production（セキュリティスタック有効時のみ。us-east-1に作成） |

アラーム発報時（`alarm_actions`）と復旧時（`ok_actions`）の両方でSNSトピックに通知する。

---

## CloudWatchダッシュボード

| 項目 | 内容 |
|------|------|
| 表示メトリクス | ECSの`RunningTaskCount`（Container Insights） |
| 目的 | Application Auto Scalingによるタスク数の増減を可視化する（負荷テスト時の証跡） |

---

## ログ

| 対象 | 出力先 | dev | staging | production |
|------|-------|-----|---------|-----------|
| ECS（Flaskアプリ） | CloudWatch Logs `/ecs/tomario-{env}` | 7日 | 30日 | 90日 |
| VPC Flow Logs | S3ログ集約バケットの`/flow-logs/` | 7日 | 30日 | 90日 |
| ALBアクセスログ | S3ログ集約バケットの`/alb/` | 7日 | 30日 | 90日 |
| WAF | CloudWatch Logs `aws-waf-logs-tomario-production-*` | ― | ― | 30日（セキュリティスタック有効時のみ） |
| RDS | 収集しない | ― | ― | ― |

- ECSのログは`awslogs-multiline-pattern`を設定し、複数行のスタックトレースを1イベントにまとめる
- CloudWatch Logsの`retention_in_days`、S3ライフサイクルの保持日数は変数で環境ごとに設定する（S3バケットの定義は[logging-environment-design.md](logging-environment-design.md)参照）

---

## 未解決事項

なし
