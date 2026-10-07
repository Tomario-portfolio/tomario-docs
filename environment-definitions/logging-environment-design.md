# logging 環境定義

ログ集約用のS3バケットを管理するコンポーネント（`modules/logging`）。
ALBアクセスログ・VPC Flow Logs・CloudTrailの書き込み先を集約する。ECSアプリケーションログ・WAFログはCloudWatch Logsに出力する（[monitoring-environment-design.md](monitoring-environment-design.md)参照）。

---

## ログ集約バケット

| 項目 | dev | staging | nonprod/shared | production |
|------|-----|---------|----------------|-----------|
| バケット名 | tomario-dev-logs-{nonprod_account_id} | tomario-staging-logs-{nonprod_account_id} | tomario-shared-cloudtrail-{nonprod_account_id} | tomario-production-logs-{prod_account_id} |
| 保持期間（ライフサイクルの`expiration`） | 7日 | 30日 | 90日 | 90日 |
| パブリックアクセスブロック | 全て有効 | 全て有効 | 全て有効 | 全て有効 |
| 暗号化 | SSE-S3（AES256、明示設定） | 同左 | 同左 | 同左 |
| 用途 | ALBアクセスログ・VPC Flow Logs | 同左 | CloudTrail | ALBアクセスログ・VPC Flow Logs・CloudTrail |

- nonprod/sharedのバケットは既存名を`bucket_name`変数で明示指定している（S3バケット名はリネーム不可のため）
- productionはshared相当の環境を持たないため、CloudTrailも同じバケットに書き込む
- 暗号化はSSE-S3とする（ALBアクセスログの配信先はSSE-KMS非対応のため）

---

## バケットポリシー（書き込み許可）

| 書き込み元 | サービスプリンシパル | 書き込み先パス | 許可する操作 |
|-----------|-------------------|--------------|------|
| ALBアクセスログ | logdelivery.elasticloadbalancing.amazonaws.com | `/alb/AWSLogs/{account_id}/*` | PutObject |
| VPC Flow Logs | delivery.logs.amazonaws.com | `/flow-logs/AWSLogs/{account_id}/*` | GetBucketAcl・PutObject |
| CloudTrail | cloudtrail.amazonaws.com | `/AWSLogs/{account_id}/*`（プレフィックス無し） | GetBucketAcl・PutObject |

3種類のステートメントは、どの環境から呼び出しても常に含める（使わないステートメントもAWSサービスプリンシパル限定のため無害として許容）。

---

## Terraform変数

| 変数名 | 説明 | デフォルト値 |
|--------|------|------------|
| env | 環境名 | - |
| log_retention_days | S3ライフサイクルの保持日数 | 7 |
| bucket_name | バケット名を明示指定する場合に使用 | null（未指定時は`tomario-{env}-logs-{account_id}`を自動生成） |

---

## 未解決事項

| 項目 | 内容 |
|------|------|
| Athenaによる横断検索 | 未導入。ログ分析の必要性が高まった場合に検討する |
