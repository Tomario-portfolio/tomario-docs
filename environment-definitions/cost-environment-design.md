# cost 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

AWS BudgetsとCost Anomaly Detectionを管理するコンポーネント。
予算超過・コスト異常を早期にメール通知することで、想定外の課金を防止する。

**2026-07-10の構成変更：** Cost Anomaly Monitorは「AWSアカウントあたり1つのみ作成可能」という制約があり、Budgetsもアカウント単位で管理するのが自然なため、workload環境（dev/staging）ごとに重複させず`envs/nonprod/shared`という専用環境に集約した（`terraform state mv`による無停止移行）。dev環境の`main.tf`からは`cost`モジュールの呼び出し自体を削除済み。

---

## AWS Budgets（nonprod/shared環境に集約）

月間のAWSコストが予算を超過した場合にメールで通知する。

| 項目 | nonprod/shared | production |
|------|-----|-----|
| バジェット名 | tomario-shared-monthly-budget | tomario-production-monthly-budget |
| バジェット種別 | COST（コストベース） | COST |
| 予算上限 | $10/月（変数で変更可） | $130/月（一般公開前の目安。公開して常時稼働に切り替えたら試算額~$165/月に合わせて見直す予定。詳細は[cost-high-level-spec.md](../basic-design/cost-high-level-spec.md)参照） |
| 集計単位 | MONTHLY | MONTHLY |
| 通知先 | 管理者メールアドレス | 管理者メールアドレス |

dev・staging環境は個別のBudgetsを持たず、nonprod/shared側を共有する。

### 通知ルール

| 通知タイミング | 条件 | 通知種別 |
|-------------|------|---------|
| 80%到達時 | 実績コストが予算の80%を超過 | ACTUAL |
| 100%到達時 | 実績コストが予算の100%を超過 | ACTUAL |

※ 予算超過後も追加通知はない（アラームは一度だけ発火）。継続的な超過を防ぐにはcost-stop等の運用で対応する。

---

## Cost Anomaly Detection（nonprod/shared環境に集約）

サービス単位でコストの異常増加を検出し、$5以上の異常を翌日メールで通知する。

### Anomaly Monitor

| 項目 | nonprod/shared | production |
|------|-----|-----|
| モニター名 | tomario-shared-anomaly-monitor | tomario-production-anomaly-monitor |
| モニター種別 | DIMENSIONAL（AWSサービス別） | DIMENSIONAL |
| ディメンション | SERVICE | SERVICE |
| 備考 | AWSアカウントあたり1つのみ作成可能なため、shared環境に集約 | 別アカウントのため独立して作成済み |

### Anomaly Subscription

| 項目 | nonprod/shared | production |
|------|-----|-----|
| サブスクリプション名 | tomario-shared-anomaly-subscription | tomario-production-anomaly-subscription |
| 通知頻度 | DAILY（翌日通知） | DAILY |
| 通知方式 | EMAIL | EMAIL |
| 通知先 | 管理者メールアドレス | 管理者メールアドレス |
| 検知しきい値 | $5以上の異常増加 | $5以上の異常増加 |

dev・staging環境は個別のAnomaly Monitor/Subscriptionを持たず、nonprod/shared側を共有する。

※ EMAIL通知はIMMEDIATEではなくDAILYまたはWEEKLYのみサポート。SNSの場合のみIMMEDIATEが使用可能。

---

## Terraform変数

| 変数名 | 説明 | デフォルト値 |
|--------|------|------------|
| env | 環境名 | - |
| alarm_email | コストアラート通知先メールアドレス | - |
| monthly_budget_usd | 月間予算上限（USD） | 10 |

---

## TBD解消事項

| 項目 | 内容 |
|------|------|
| production環境の月間予算上限 | $130/月に決定（2026-08-03、一般公開前の目安として設定。常時稼働への切り替え時に試算額~$165/月に合わせて見直す） |
| RDS負荷テスト時の一時スケールアップコスト | Budgetの予算上限には含めていない。頻度・時間が限定的なため個別に監視する |
