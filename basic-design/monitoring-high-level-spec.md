# モニタリング

## 基本方針

システムの異常をCloudWatchアラームで検知し、SNS経由で管理者へ通知する。
インフラレベル（ALB・ECS・RDS）の監視を行い、障害の早期発見と対応を可能にする。
ログは用途ごとに集約先を分け、環境の重要度に応じた期間保持する。

## 監視対象

| 監視対象 | 監視する観点 | 対象環境 |
|---------|------------|---------|
| ALB | 異常なターゲット（ECSタスク障害）の発生 | 全環境 |
| ECS | CPU高負荷 | 全環境 |
| RDS | CPU高負荷 | 全環境 |
| WAF（CloudFront） | ブロック件数の急増（攻撃・誤検知） | production（セキュリティスタック有効時のみ） |

アラーム発報時と復旧時の両方で通知する。
メトリクス名・閾値・評価期間は[環境定義書](../environment-definitions/monitoring-environment-design.md)を参照。

## 可視化

staging・productionでは、ECSのタスク数を表示するCloudWatchダッシュボードを作成し、Auto Scalingによるタスク数の増減を可視化する。

## 通知

CloudWatchアラームの通知先としてSNSトピックを環境ごとに作成し、管理者へメールで通知する。

## ログ設計

| 対象 | 集約先 |
|------|-------|
| ECS（Flaskアプリ） | CloudWatch Logs |
| ALBアクセスログ | S3（ログ集約バケット） |
| VPC Flow Logs | S3（ログ集約バケット） |
| CloudTrail | S3（ログ集約バケット。詳細は[security-high-level-spec.md](security-high-level-spec.md)参照） |
| WAF | CloudWatch Logs（productionでセキュリティスタック有効時のみ） |

- アプリケーションログは、複数行のスタックトレースが分断されないよう1イベントにまとめて収集する
- 保持期間は環境ごとに段階を分ける。短期間で作り直すdevは短く、障害試験の分析に使うstagingは中程度、気づくのが遅れたインシデントも追えるようproductionは最も長くする

ロググループ名・バケット名・保持日数の具体値は[環境定義書](../environment-definitions/logging-environment-design.md)（[monitoring](../environment-definitions/monitoring-environment-design.md)も参照）に記載する。

## 性能分析

RDSのクエリ単位の性能分析（Performance Insights）は導入しない。経緯は[ADR: RDS Performance Insightsの導入見送り](../../tomario-steering/adr/infra/database/004-performance-insights-instance-class-limitation.md)を参照。
