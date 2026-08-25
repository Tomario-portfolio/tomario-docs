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
| 証跡名 | tomario-shared-trail | tomario-production-trail（予定） |
| 対象イベント | 管理イベントのみ（データイベントは対象外） | 管理イベントのみ |
| グローバルサービスイベント | 含む（IAM操作など） | 含む |
| マルチリージョン証跡 | 無効（東京リージョンのみ） | 無効 |
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
| 有効化 | 有効 | 有効化予定 |
| 検出器名（タグ） | tomario-shared-guardduty | tomario-production-guardduty（予定） |
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
| S3バケット名 | tomario-shared-config-{nonprod_account_id} | tomario-production-config-{prod_account_id}（予定） |

---

## WAF（2026-07-11：導入する方針に変更、2026-08-03：面接期間のみ有効化に方針詳細化）

CloudFront・ALB双方にAWS Managed Rule Groupsを使ったWAF Web ACLを導入する方針。

| 項目 | dev / staging | production |
|------|-----|-----|
| 導入方針 | 導入しない | 導入する（cost-stopと同じ発想で、常時起動はせず面接が近いタイミングだけ作成） |
| 理由 | 検証環境のためコスト対効果が薄い | 実運用環境として保護は必要だが、常時起動コスト（月$14〜16）は非商用ポートフォリオでは過大 |
| Managed Rule Groups | ― | `AWSManagedRulesCommonRuleSet`（OWASP Top10相当の汎用防御）、`AWSManagedRulesKnownBadInputsRuleSet`（既知の悪意あるリクエストパターン）、`AWSManagedRulesSQLiRuleSet`（DBバックエンドのためSQLi対策を追加）の3つをCloudFront用Web ACLにアタッチ |

**判断の経緯：** 当初は非商用ポートフォリオのため見送っていたが、WAFのWeb ACLは「停止」ができず「存在（課金）／削除（無課金）」の二択かつ時間按分課金であることが判明。cost-stop/startと同じ発想でWeb ACLも使う時だけ作成・削除する運用にすれば実質数十〜百円/月に収まるため、導入する方針に変更した（常時起動なら月$14〜16程度）。production自体はリリース後常時稼働だが、WAFは常時稼働に切り替えた後も面接が近いタイミングだけ作成する運用とする（2026-08-03決定）。実装（cost-stop.yml/cost-start.ymlへの組み込み含む）はproduction構築のタイミングで対応する。

---

## 有効化フラグ（Terraform変数）

| 変数名 | nonprod/shared | production |
|--------|-----|-----|
| enable_security_hub | false | 面接期間のみtrue、それ以外はfalse |
| enable_config | false | 面接期間のみtrue、それ以外はfalse |

---

## TBD解消事項

なし（2026-08-03、Security Hub/Config有効化タイミングとWAF Managed Rule Group選定を決定）
