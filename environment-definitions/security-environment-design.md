# security 環境定義

CloudTrail・GuardDuty・Security Hub・AWS Config（`modules/security`）とWAF（`modules/waf`）を定義する。
いずれもアカウント単位で管理するため、nonprodは`envs/nonprod/shared`、prodは`envs/prod/production/security`で作成する（dev・stagingは個別に持たず、nonprod/sharedを共有する）。

WAF・Security Hub・AWS Configはproductionのみ導入し、必要な期間だけ有効化する。理由は[ADR: セキュリティスタックの「使う時だけ有効化」運用](../../tomario-steering/adr/infra/security/001-security-stack-cost-management.md)を参照。

---

## CloudTrail

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 証跡名 | tomario-shared-trail | tomario-production-trail |
| 対象イベント | 管理イベントのみ（データイベントは対象外） | 管理イベントのみ |
| グローバルサービスイベント | 含む | 含む |
| マルチリージョン証跡 | 有効（`is_multi_region_trail=true`） | 有効 |
| ログファイル整合性検証 | 有効 | 有効 |
| S3バケット名 | tomario-shared-cloudtrail-{nonprod_account_id} | tomario-production-logs-{prod_account_id} |
| S3保持期間 | 90日 | 90日 |

S3バケットは`modules/logging`で管理する（[logging-environment-design.md](logging-environment-design.md)参照）。

---

## GuardDuty

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 有効化 | 常時有効 | 常時有効 |
| 検出器名（タグ） | tomario-shared-guardduty | tomario-production-guardduty |

---

## Security Hub

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 有効化 | 無効（コードのみ） | 必要な期間のみ有効化 |
| 有効化する標準 | ― | CIS AWS Foundations Benchmark v1.4.0 |
| 前提条件 | ― | AWS Config・CloudTrail |

---

## AWS Config

| 項目 | nonprod/shared | production |
|------|-----|-----|
| 有効化 | 無効（コードのみ） | 必要な期間のみ有効化（Security Hubと同時） |
| 記録対象 | ― | 全リソース |
| S3バケット名 | tomario-shared-config-{nonprod_account_id} | tomario-production-config-{prod_account_id} |
| S3暗号化 | SSE-S3（AES256、明示設定） | SSE-S3（AES256、明示設定） |

---

## WAF

| 項目 | dev / staging | production |
|------|-----|-----|
| 導入 | なし | 必要な期間のみ作成 |
| Web ACL | ― | CloudFront用（us-east-1、CLOUDFRONTスコープ）・ALB用（ap-northeast-1、REGIONALスコープ） |
| Managed Rule Groups | ― | `AWSManagedRulesCommonRuleSet`・`AWSManagedRulesKnownBadInputsRuleSet`・`AWSManagedRulesAmazonIpReputationList` |
| レートベースルール | ― | `rate-limit`（上限は`rate_limit`変数で指定） |
| ログ出力 | ― | CloudWatch Logs（[monitoring-environment-design.md](monitoring-environment-design.md)参照） |

SQLi専用ルール（`AWSManagedRulesSQLiRuleSet`）は適用しない。

---

## 有効化フラグ

| 変数名 | nonprod/shared | production |
|--------|-----|-----|
| enable_security_hub | false | `enable_security_stack`に連動 |
| enable_config | false | `enable_security_stack`に連動 |
| enable_waf | ― | `enable_security_stack`に連動 |

productionでは環境側の`enable_security_stack`フラグ1つで3つをまとめて切り替える。切り替えは`tomario-infra`の`security-stack.yml`で行い、cost-stop/startとは独立させる。

---

## アカウントレベルの保険設定

| 設定 | nonprod/shared | production |
|------|-----|-----|
| EBSスナップショットのパブリックアクセスブロック | 有効 | 有効 |
| SSM Documentsのパブリック共有ブロック | 有効 | 有効 |

---

## 未解決事項

なし
