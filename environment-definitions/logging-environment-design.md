# logging 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | >= 1.10 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

ログ集約用のS3バケットを管理するコンポーネント（`modules/logging`）。

**2026-07-13の新設経緯：** ECSログ（`backend`）・VPC Flow Logs（`network`）・CloudTrail（`security`）がそれぞれのモジュールにバラバラに実装されており、集約先・保持方針を横断的に設計したコンポーネントが存在しないことが判明した。ALB・VPC Flow Logs・CloudTrailの書き込み先を1つのS3バケットに集約する「中間版」スコープで新設（Athenaによる横断検索等のフル版は見送り）。

---

## ログ集約バケット（dev / staging環境ごとに新規作成）

| 項目 | dev | staging |
|------|-----|---------|
| バケット名 | tomario-dev-logs-{nonprod_account_id} | tomario-staging-logs-{nonprod_account_id} |
| 保持期間（ライフサイクルの`expiration`） | 7日 | 30日 |
| パブリックアクセスブロック | 全て有効 | 全て有効 |
| 用途 | ALBアクセスログ・VPC Flow Logs | 同左 |

workload環境（dev/staging）ごとに独立したバケットを持つ（`network`・`backend`と同じ「環境ごとに独立」の設計方針に合わせる）。

---

## ログ集約バケット（nonprod/shared環境、CloudTrail用）

| 項目 | nonprod/shared |
|------|-----|
| バケット名 | tomario-shared-cloudtrail-{nonprod_account_id}（**既存名を維持**） |
| 保持期間 | 90日 |
| 用途 | CloudTrailのみ |

**既存バケットをそのまま引き継ぐ理由：** CloudTrail用バケットは`security-environment-design.md`の通り元々`modules/security`が単独で管理していたが、ログ管理の一本化にあたり`modules/logging`に統合した（2026-07-13、`terraform state mv`による無停止移行）。S3バケット名はAWSでリネーム不可（ForceNew）なため、`modules/logging`の`bucket_name`変数で既存名を明示的に指定し、destroy/createを発生させないようにしている（ECRの命名継承と同じ考え方）。

sharedはVPCを持たない（`network-environment-design.md`参照）ため、このバケットはALBアクセスログ・VPC Flow Logsの用途では使われない（バケットポリシー自体は共通で持つが、未使用の権限は無害として許容している）。

---

## バケットポリシー（書き込み許可）

| 書き込み元 | サービスプリンシパル | 書き込み先パス | 備考 |
|-----------|-------------------|--------------|------|
| ALBアクセスログ | logdelivery.elasticloadbalancing.amazonaws.com | `/alb/AWSLogs/{account_id}/*` | AWS Log Delivery方式。リージョン別ELBアカウント方式（`aws_elb_service_account`）ではAccess Deniedになったため採用（2026-07-13） |
| VPC Flow Logs | delivery.logs.amazonaws.com | `/flow-logs/AWSLogs/{account_id}/*` | GetBucketAcl（AclCheck）とPutObject（Write）の2ステートメント |
| CloudTrail | cloudtrail.amazonaws.com | `/AWSLogs/{account_id}/*`（プレフィックス無し、既存パスを維持） | GetBucketAcl（AclCheck）とPutObject（Write）の2ステートメント |

3種類の書き込み元すべてのステートメントを、dev/staging/sharedいずれの呼び出しでも常に含める（呼び出し元ごとに使わないステートメントがあっても、AWSサービスプリンシパル限定で実害が無いため、トグル変数は設けず単純化している）。

---

## Terraform変数

| 変数名 | 説明 | デフォルト値 |
|--------|------|------------|
| env | 環境名 | - |
| log_retention_days | S3ライフサイクルの保持日数 | 7 |
| bucket_name | バケット名を明示指定する場合に使用（既存バケットを引き継ぐ場合など） | null（未指定時は`tomario-{env}-logs-{account_id}`を自動生成） |

---

## production

dev/stagingと同様に独立したバケット（`tomario-production-logs-{prod_account_id}`）を新規作成する。保持期間は90日（ログ保持方針、`monitoring-high-level-spec.md`参照）。用途もdev/staging同様、ALBアクセスログ・VPC Flow Logsの集約先とする。CloudTrail用バケットはprod用に別アカウントで新規作成されるため、既存バケット引き継ぎの考慮は不要（nonprod/sharedのケースとは異なる）。production環境はshared相当のディレクトリを持たないため、CloudTrail用バケットも`envs/prod/production`配下でこの`modules/logging`呼び出しに含めて管理する。

---

## TBD解消事項

| 項目 | 内容 |
|------|------|
| Athenaによる横断検索 | 今回は「中間版」スコープのため見送り。将来的にログ分析の必要性が高まった場合に検討する |
| production環境のALBアクセスログ | dev/stagingと同様、S3の`/alb/`プレフィックスへ出力する（決定済み）。CloudFront→ALB間のプロトコル（現状HTTP、`X-Origin-Verify`ヘッダーで検証）は独自ドメイン導入とは独立した話のため、アクセスログの方針とは切り離して扱う |
