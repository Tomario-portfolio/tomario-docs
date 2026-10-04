# security 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

CloudTrail・GuardDuty・Security Hub・AWS Config・WAFを管理するコンポーネント。
CIS AWS Foundations Benchmarkへの準拠を将来的な目標とし、段階的に導入する。

**2026-07-10の構成変更：** CloudTrail・GuardDuty・Budgetsはアカウント/リージョンにつき1つまでしか作れない（またはアカウント単位で管理するのが自然な）シングルトンリソースであるため、workload環境（dev/staging）ごとに重複させず、`envs/nonprod/shared`という専用環境に集約した（`terraform state mv`による無停止移行）。dev環境の`main.tf`からは`security`/`cost`モジュールの呼び出し自体を削除済み。

---

## CloudTrail（nonprod/shared環境に集約）

AWSアカウント上の全API操作を記録し監査証跡を保存する。

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 証跡名 | tomario-shared-trail | tomario-production-trail |
| 対象イベント | 管理イベントのみ（データイベントは対象外） | 管理イベントのみ |
| グローバルサービスイベント | 含む（IAM操作など） | 含む |
| マルチリージョン証跡 | 有効（`is_multi_region_trail=true`、2026-09-16修正。WAF用CloudFront Web ACL等us-east-1で発生するAPI操作を記録するため） | 有効（同左） |
| ログファイル整合性検証 | 有効 | 有効 |
| S3バケット名 | tomario-shared-cloudtrail-{nonprod_account_id} | tomario-production-logs-{prod_account_id} |
| S3保持期間 | 90日（ライフサイクルポリシー） | 90日 |
| 追加コスト | なし（管理イベントは無料） | なし |

dev・staging環境は個別のCloudTrailを持たず、nonprod/shared側を共有する（アカウント単位で1つのため）。

S3バケット自体は2026-07-13より`modules/logging`が管理する（詳細は`logging-environment-design.md`参照）。nonprod/sharedは既存バケット（`tomario-shared-cloudtrail-*`）をリネーム不可のため`bucket_name`変数で明示指定して引き継いでいるが、production環境はshared相当のディレクトリを持たないため、CloudTrail専用バケットを別途作らず、ALBアクセスログ・VPC Flow Logsと同じ`tomario-production-logs-{prod_account_id}`バケットに`AWSLogs/`プレフィックスで書き込む（新規構築のためリネーム制約による引き継ぎの考慮は不要）。

---

## GuardDuty（nonprod/shared環境に集約）

AWSアカウントへの脅威を継続的に検出するマネージドサービス。

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 有効化 | 有効 | 有効 |
| 検出器名（タグ） | tomario-shared-guardduty | tomario-production-guardduty |
| 試用後コスト | ~$0.50/月（低トラフィック） | トラフィックに応じて変動 |

dev・staging環境は個別のGuardDutyを持たず、nonprod/shared側を共有する（アカウント/リージョンにつき1つまでのため）。

---

## Security Hub（nonprod/sharedでは無効、production構築時に導入・面接期間のみ有効化）

CIS AWS Foundations Benchmarkへの準拠状況を可視化する。

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 有効化 | 無効（コードのみ実装済み） | コードは常設、**有効化は面接期間のみ**（`enable_security_hub`をtrue/falseで切り替え） |
| CIS標準 | 無効 | CIS AWS Foundations Benchmark v1.4.0 |
| 前提条件 | CloudTrail（対応済み）・AWS Config（未対応） | 同上（AWS Configも同じく面接期間のみ有効化） |

Security Hubの有効化にはAWS Configが必要。
非商用のnonprod環境でCIS準拠チェックまで行う必要性は薄いと判断し、nonprod/sharedでは無効のままとする（軽量版）。production環境はリリース後常時稼働だが、Security Hub/Configは常時起動コスト（~$3〜5/月）に見合わないと判断し、面接が近いタイミングだけ有効化する運用とする（2026-08-03決定、詳細は`cost-high-level-spec.md`参照）。

---

## AWS Config（nonprod/sharedでは無効、production構築時に導入・面接期間のみ有効化）

AWSリソースの設定変更履歴を記録しコンプライアンス評価を行う。

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 有効化 | 無効（コードのみ実装済み） | コードは常設、**有効化は面接期間のみ**（Security Hubと同時に切り替え） |
| 記録対象 | 全リソース | 全リソース |
| 月額コスト目安 | ~$1〜3（記録件数による） | 有効化時のみ課金（面接期間外は$0） |
| S3バケット名 | tomario-shared-config-{nonprod_account_id} | tomario-production-config-{prod_account_id} |

---

## WAF（2026-07-11：導入する方針に変更、2026-08-03：面接期間のみ有効化に方針詳細化）

CloudFront・ALB双方にAWS Managed Rule Groupsを使ったWAF Web ACLを導入する方針。

| 項目 | dev / staging | production |
|------|-----|-----|
| 導入方針 | 導入しない | 導入する（cost-stopと同じ発想で、常時起動はせず面接が近いタイミングだけ作成） |
| 理由 | 検証環境のためコスト対効果が薄い | 実運用環境として保護は必要だが、常時起動コスト（月$14〜16）は非商用ポートフォリオでは過大 |
| Managed Rule Groups | ― | `AWSManagedRulesCommonRuleSet`（OWASP Top10相当の汎用防御）、`AWSManagedRulesKnownBadInputsRuleSet`（既知の悪意あるリクエストパターン）、`AWSManagedRulesAmazonIpReputationList`（不審IPのブロック）の3つ＋レートベースルール（`rate_limit`、DoS・ブルートフォース対策）をアタッチ。SQLi専用ルール（`AWSManagedRulesSQLiRuleSet`）は導入していない（非機能試験S-07で対象外と確認済み） |

**判断の経緯：** [ADR: セキュリティスタックの「使う時だけ有効化」運用](../../tomario-steering/adr/infra/security/001-security-stack-cost-management.md)を参照。

---

## 有効化フラグ（Terraform変数）

| 変数名 | nonprod/shared | production |
|--------|-----|-----|
| enable_security_hub | false | 面接期間のみtrue、それ以外はfalse |
| enable_config | false | 面接期間のみtrue、それ以外はfalse |

`modules/security`自体はこの2変数を別々に持つが、`envs/prod/production/security`では両方とも環境側の`enable_security_stack`フラグ1つに連動させている（WAFの`enable_waf`も同じフラグ）。3つをまとめて`security-stack.yml`から切り替えられるようにするため（詳細は[security-high-level-spec.md](../basic-design/security-high-level-spec.md)参照）。

---

## TBD解消事項

なし（2026-08-03、Security Hub/Config有効化タイミングとWAF Managed Rule Group選定を決定）
