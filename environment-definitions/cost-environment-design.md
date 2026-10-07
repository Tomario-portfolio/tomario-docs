# cost 環境定義

AWS BudgetsとCost Anomaly Detectionを管理するコンポーネント（`modules/cost`）と、リソース別のコスト目安を定義する。
方針は[cost-high-level-spec.md](../basic-design/cost-high-level-spec.md)、cost-stop/start運用を採った理由は[ADR: cost-stop/startによる使う時だけ起動する運用](../../tomario-steering/adr/infra/cost/001-cost-stop-start-operation.md)を参照。

Budgets・Cost Anomaly Monitorはアカウント単位のリソースのため、nonprodは`envs/nonprod/shared`に1つだけ作成し、dev・stagingで共有する。prodは`envs/prod/production/cost`で作成する。

---

## AWS Budgets

| 項目 | nonprod/shared | production |
|------|-----|-----|
| バジェット名 | tomario-shared-monthly-budget | tomario-production-monthly-budget |
| バジェット種別 | COST | COST |
| 予算上限 | $10/月 | $130/月 |
| 集計単位 | MONTHLY | MONTHLY |
| 通知先 | 管理者メールアドレス | 管理者メールアドレス |

### 通知ルール

| 通知タイミング | 条件 | 通知種別 |
|-------------|------|---------|
| 80%到達時 | 実績コストが予算の80%を超過 | ACTUAL |
| 100%到達時 | 実績コストが予算の100%を超過 | ACTUAL |

予算超過後の追加通知はない（各しきい値で一度だけ通知）。

---

## Cost Anomaly Detection

### Anomaly Monitor

| 項目 | nonprod/shared | production |
|------|-----|-----|
| モニター名 | tomario-shared-anomaly-monitor | tomario-production-anomaly-monitor |
| モニター種別 | DIMENSIONAL | DIMENSIONAL |
| ディメンション | SERVICE | SERVICE |

### Anomaly Subscription

| 項目 | nonprod/shared | production |
|------|-----|-----|
| サブスクリプション名 | tomario-shared-anomaly-subscription | tomario-production-anomaly-subscription |
| 通知頻度 | DAILY | DAILY |
| 通知方式 | EMAIL | EMAIL |
| 通知先 | 管理者メールアドレス | 管理者メールアドレス |
| 検知しきい値 | $5以上の異常増加 | $5以上の異常増加 |

EMAIL通知はDAILYまたはWEEKLYのみ指定可能（IMMEDIATEはSNS通知のみ）。

### 日次の異常検知しきい値

| 項目 | nonprod | production |
|------|-----|-----|
| しきい値 | $0.24/日 | $0.24/日（nonprodの実測値を暫定流用） |

cost-stop状態でのベースライン実測（約$0.23/日）をわずかに上回る値とする。

---

## Terraform変数

| 変数名 | 説明 | デフォルト値 |
|--------|------|------------|
| env | 環境名 | - |
| alarm_email | コストアラート通知先メールアドレス | - |
| monthly_budget_usd | 月間予算上限（USD） | 10 |

---

## リソース別コスト目安

試算額は東京リージョンの単価に基づく概算であり、実績値ではない。

### 無料で使えるリソース

| リソース | 理由 |
|---------|------|
| VPC / サブネット / IGW / ルートテーブル | 完全無料 |
| セキュリティグループ | 完全無料 |
| IAM | 完全無料 |
| CloudWatch メトリクス・アラーム（10個まで） | 無料枠内 |
| SNS | 100万通知/月まで無料 |
| GitHub Actions | パブリックリポジトリは無制限無料 |
| S3 VPCエンドポイント（Gateway型） | 完全無料 |
| CloudTrail（管理イベント） | 完全無料 |
| AWS Budgets（2つまで） | 完全無料 |
| Cost Anomaly Detection | 完全無料 |

### コストが発生するリソースと対策後のコスト

| リソース | 常時起動した場合の月額目安 | 対策 | 対策後のコスト |
|---------|----------|------|-------------|
| ALB | ~$18/月 | 作業時以外は削除する | $0 |
| ECS Fargate | ~$9/月/タスク（0.25vCPU / 0.5GB、ARM64） | 作業時以外はタスク数0にする | $0 |
| VPCエンドポイント（Interface型×5、2AZ） | ~$102/月（$0.014/時間 × ENI 10個） | 作業時以外は削除する | $0 |
| RDS（db.t3.micro、Single-AZ） | ~$22/月（インスタンス ~$19 + ストレージ20GB ~$3） | 作業時以外は停止する | ~$3/月（ストレージのみ） |
| Secrets Manager | ~$0.80/月 | DB認証情報・Flask SECRET_KEYの2件を管理 | ~$0.80/月 |
| GuardDuty | ~$1〜3/月 | 停止できないため常時有効 | ~$0.50/月（低トラフィック） |
| VPC Flow Logs（S3出力） | ~$0.02/月 | ― | ~$0.02/月 |
| CloudWatch（Container Insights） | 稼働時間に応じた時間按分 | 稼働時のみメトリクスが発生 | 月10時間稼働で~$0.15/月 |

### 使用しないリソース

| リソース | 月額 | 不使用の理由 |
|---------|------|------------|
| NAT ゲートウェイ | ~$45/月（AZごと） | VPCエンドポイントで代替 |

### 必要な期間だけ有効化するリソース（productionのみ）

| リソース | 常時起動した場合の月額目安 | 運用 |
|---------|-------------------------|---------|
| AWS WAF（CloudFront・ALB用Web ACL） | ~$14〜16/月 | 必要な期間のみ作成、それ以外は削除 |
| Security Hub + AWS Config | ~$3〜5/月 | 必要な期間のみ有効化、それ以外は無効化 |

RDSの負荷テスト時の一時スケールアップ（`db.t4g.medium`）のコストは、頻度・時間が限定的なため予算上限には含めていない。

---

## 運用状況別の試算

### dev / staging

| 状況 | 月額目安 |
|------|---------|
| 普段（ALB削除・ECSタスク0・VPCエンドポイント削除・RDS停止） | ~$7/月（nonprodアカウント全体の実測 約$0.23/日） |
| 作業中（すべて起動） | 時間課金。1時間あたり~$0.2（dev）、~$0.22（staging、タスク2つ） |

### production

| 状況 | 月額目安 |
|------|---------|
| 普段（公開前・cost-stop中） | ~$7/月（nonprodと同程度の想定） |
| 稼働時（作業確認・試験など） | 時間課金。1時間あたり~$0.22 |
| 常時稼働（公開後） | ~$165/月（内訳は下表） |

#### 常時稼働（公開後）の内訳

| リソース | 月額目安 | 備考 |
|---------|---------|------|
| ALB | ~$18/月 | |
| ECS Fargate（0.25vCPU/0.5GB×2タスク、ARM64） | ~$18/月 | オートスケール時（最大4タスク）はさらに増加 |
| VPCエンドポイント（Interface×5、2AZ） | ~$102/月 | 最大のコスト要因（全体の約6割） |
| RDS（db.t3.micro、Single-AZ） | ~$22/月 | |
| Secrets Manager | ~$0.80/月 | DB認証情報・SECRET_KEYの2件 |
| GuardDuty | ~$1〜3/月 | |
| CloudTrail（管理イベント） | $0 | |
| VPC Flow Logs（S3出力） | ~$0.02/月 | |
| **常時稼働ベースライン合計** | **~$165/月** | WAF・Security Hub・Configは含まない |

---

## 未解決事項

| 項目 | 内容 |
|------|------|
| productionの月間予算 | 現在の$130/月は一般公開前の目安。常時稼働へ切り替える時点で試算額（~$165/月）に合わせて見直す |
| productionの日次異常検知しきい値 | nonprodの実測値を暫定流用している。production単独のcost-stop状態のベースラインを実測して見直す |
