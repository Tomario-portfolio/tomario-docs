# コスト設計

## 1. 基本方針

全環境で、コストのかかるリソースは「使うときだけ起動する」運用で対応する（cost-stop/start）。productionも一般公開前は同じ運用とし、公開時に常時稼働へ切り替える。選定理由は[ADR: cost-stop/startによる使う時だけ起動する運用](../../tomario-steering/adr/infra/cost/001-cost-stop-start-operation.md)を参照。

"停止"ができないが"作成・削除"や"一時変更"は可能なリソースについても、同じ発想で「使う時だけ作る」運用に寄せる。

リソース別の月額目安・運用状況別の試算は[環境定義書](../environment-definitions/cost-environment-design.md)に記載する。

---

## 2. リソース別の方針

| 分類 | 対象リソース | 方針 |
|------|------------|------|
| 停止・削除できる時間課金リソース | ALB・ECSタスク・Interface型VPCエンドポイント・RDS | 未使用時はALB・VPCエンドポイントを削除、ECSタスク数を0、RDSを停止する（cost-stop対象） |
| 停止できず、作成・削除で代替するリソース | WAF（Web ACL）・AWS Config・Security Hub | productionのみ導入し、必要な期間だけ作成・有効化する（[security-high-level-spec.md](security-high-level-spec.md)参照） |
| 一時変更で代替するリソース | RDSインスタンスクラス | 負荷テスト時のみ一時的にスケールアップし、終了後に戻す（[ADR: RDSインスタンスクラスの選定](../../tomario-steering/adr/infra/database/005-db-instance-class-selection.md)参照） |
| 常時有効とするリソース | GuardDuty・CloudTrail（管理イベント）・Secrets Manager・VPC Flow Logs | 停止できない、または低額なため常時有効 |
| 使用しないリソース | NAT Gateway | VPCエンドポイントで代替する（[network-high-level-spec.md](network-high-level-spec.md)参照） |

---

## 3. コスト管理の運用

### cost-stop / cost-start

`tomario-infra`の`cost-stop.yml`・`cost-start.yml`（GitHub Actions）で、上記の停止・削除対象リソースをまとめて削除/作成・停止/起動する。対象アカウント（nonprod/prod）はワークフローの入力で選択する。

### RDSの7日自動起動への対策

RDSは停止しても7日後にAWSにより自動で起動される。
全環境にRDS自動停止Lambda（`modules/rds-autostop`）を配置し、EventBridgeで1時間ごとに起動して、7日制約による自動起動のRDSイベントが記録され、かつECSサービスが稼働していない場合に再停止する。cost-startや手動で起動した場合は自動起動のイベントが記録されないため、止めない。

### コスト監視

アカウントごとに以下で想定外の課金を早期に検知する。

| 仕組み | 目的 |
|-------|------|
| AWS Budgets（月間予算） | 月間予算の到達をメール通知する |
| 日次の異常検知 | cost-stop状態のベースラインをわずかに上回る閾値とし、cost-stopの消し忘れを翌日には検知する |
| Cost Anomaly Detection | サービス別の異常な増加をメール通知する |

予算額・閾値の具体値は[環境定義書](../environment-definitions/cost-environment-design.md)を参照。

### productionの公開時の切り替え

公開時は常時稼働へ切り替える。必要な変更は以下の通り。

- cost-stop/startの運用対象からproductionを外す
- ALB削除保護を有効化する（現在はcost-stopでALBを削除するため全環境無効）
- 月間予算を常時稼働の試算額に合わせて見直す
- 常時稼働時に最大のコスト要因となるVPCエンドポイントの削減策（ENIを1AZに寄せる、利用頻度の低いエンドポイントを必要時のみ作成する等）を検討する
