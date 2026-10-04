# ネットワーク

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

## リージョン・環境

リージョン：ap-northeast-1（東京）

環境は nonprodアカウント（`dev` / `staging` / `shared`）と prodアカウント（`production`）で構成する。
workload環境（dev/staging/production）ごとに独立したVPCを作成する。`shared`（GuardDuty/CloudTrail/Budgets/ECR等のアカウント単位リソース）はVPCを持たない。
将来のVPCピアリングに備え、環境間でCIDRが重複しない設計とする。

## VPC

| 項目 | dev | staging | production |
|------|-----|---------|---|
| VPC名 | tomario-dev-vpc | tomario-staging-vpc | tomario-production-vpc |
| CIDR | 10.0.0.0/16 | 10.1.0.0/16 | 10.2.0.0/16 |
| DNSサポート | 有効 | 有効 | 有効 |
| DNSホスト名 | 有効 | 有効 | 有効 |

## Availability Zone

ap-northeast-1a と ap-northeast-1c の2AZ構成とする。

## サブネット

### dev環境

| サブネット名 | 種別 | AZ | CIDR | 配置リソース | パブリックIP自動割り当て |
|------------|------|----|------|------------|-------------------|
| tomario-dev-public-ap-northeast-1a | パブリック | ap-northeast-1a | 10.0.0.0/24 | ALB | 有効 |
| tomario-dev-public-ap-northeast-1c | パブリック | ap-northeast-1c | 10.0.1.0/24 | ALB | 有効 |
| tomario-dev-private-ap-northeast-1a | プライベート | ap-northeast-1a | 10.0.10.0/24 | ECSタスク・RDS | 無効 |
| tomario-dev-private-ap-northeast-1c | プライベート | ap-northeast-1c | 10.0.11.0/24 | ECSタスク・RDS | 無効 |

### staging環境

| サブネット名 | 種別 | AZ | CIDR | 配置リソース | パブリックIP自動割り当て |
|------------|------|----|------|------------|-------------------|
| tomario-staging-public-ap-northeast-1a | パブリック | ap-northeast-1a | 10.1.0.0/24 | ALB | 有効 |
| tomario-staging-public-ap-northeast-1c | パブリック | ap-northeast-1c | 10.1.1.0/24 | ALB | 有効 |
| tomario-staging-private-ap-northeast-1a | プライベート | ap-northeast-1a | 10.1.10.0/24 | ECSタスク・RDS | 無効 |
| tomario-staging-private-ap-northeast-1c | プライベート | ap-northeast-1c | 10.1.11.0/24 | ECSタスク・RDS | 無効 |

devと同一構成（`network`モジュールを`env="staging"`で呼ぶのみ）。CIDR以外の差分は無い。

### production環境

| サブネット名 | 種別 | AZ | CIDR | 配置リソース | パブリックIP自動割り当て |
|------------|------|----|------|------------|-------------------|
| tomario-production-public-ap-northeast-1a | パブリック | ap-northeast-1a | 10.2.0.0/24 | ALB | 有効 |
| tomario-production-public-ap-northeast-1c | パブリック | ap-northeast-1c | 10.2.1.0/24 | ALB | 有効 |
| tomario-production-private-ap-northeast-1a | プライベート | ap-northeast-1a | 10.2.10.0/24 | ECSタスク・RDS | 無効 |
| tomario-production-private-ap-northeast-1c | プライベート | ap-northeast-1c | 10.2.11.0/24 | ECSタスク・RDS | 無効 |

dev/stagingと同一構成（`network`モジュールを`env="production"`で呼ぶのみ）。CIDR以外の差分は無い。

## Internet Gateway

| 項目 | dev | staging | production |
|------|-----|---------|---|
| IGW名 | tomario-dev-igw | tomario-staging-igw | tomario-production-igw |

## ルートテーブル

| ルートテーブル名 | 関連サブネット | ルート |
|--------------|-------------|--------|
| tomario-dev-public-rt | dev パブリックサブネット×2 | 0.0.0.0/0 → IGW |
| tomario-dev-private-rt | dev プライベートサブネット×2 | VPC内通信のみ（local） |
| tomario-staging-public-rt | staging パブリックサブネット×2 | 0.0.0.0/0 → IGW |
| tomario-staging-private-rt | staging プライベートサブネット×2 | VPC内通信のみ（local） |
| tomario-production-public-rt | production パブリックサブネット×2 | 0.0.0.0/0 → IGW |
| tomario-production-private-rt | production プライベートサブネット×2 | VPC内通信のみ（local） |

## NAT Gateway

| 環境 | 方針 |
|------|------|
| dev / staging | 使用しない（VPCエンドポイントで代替） |
| production | 使用しない（同上） |

唯一の必要理由だった`bootstrap_image`のpublic ECR Gallery参照は、プライベートECR参照に修正済み（2026-07-10、詳細はcompute側参照）。

## VPC Flow Logs

VPC内の全ネットワークトラフィックを記録する。

| 項目 | dev | staging | production |
|------|-----|---------|---|
| リソース名 | tomario-dev-flow-log | tomario-staging-flow-log | tomario-production-flow-log |
| トラフィック種別 | ALL（許可・拒否を両方記録） | ALL | ALL |
| 送信先 | S3（ログ集約バケット、`logging-environment-design.md`参照） | 同左 | 同左 |
| 出力先パス | `/flow-logs/AWSLogs/{account_id}/*` | 同左 | 同左 |
| 保持期間 | 7日（S3ライフサイクル） | 30日（ログ保持方針に準ずる） | 90日（ログ保持方針に準ずる） |
| IAMロール | 不要（S3配信はバケットポリシーで許可、IAMロールを介さない） | 同左 | 同左 |

**2026-07-13変更：** 元はCloudWatch Logsへ送信していたが、ログ管理の一本化に伴いS3直接出力へ変更。専用のIAMロール（`tomario-{env}-flow-log-role`）・CloudWatch Logsグループは不要になり削除した。

## VPCエンドポイント

ECSタスク（プライベートサブネット）がAWSサービスに接続するために使用する。
dev・staging環境では未使用時に削除してコストを削減する（cost-stop対象）。
production環境はリリース前、dev/stagingと同じcost-stop対象とする。リリース後は常時稼働へ切り替え、常に作成された状態を維持する（NAT Gatewayを使わない設計のため、このエンドポイント群が無いとECSタスクがAWSサービスへ到達できない）。

### VPCエンドポイント-SG

| 項目 | dev | staging | production |
|------|-----|---------|---|
| SG名 | tomario-dev-vpce-sg | tomario-staging-vpce-sg | tomario-production-vpce-sg |

| 方向 | プロトコル | ポート | 送信元 |
|------|----------|-------|--------|
| インバウンド | TCP | 443 | ECS-SG |
| アウトバウンド | 全て | 全て | 0.0.0.0/0 |

### エンドポイント一覧

| エンドポイント名 | サービス | 種別 | dev | staging | production |
|--------------|---------|------|-----|---------|---|
| tomario-{env}-ecr-api-vpce | ECR API | Interface | あり | あり | あり（リリース後は常時） |
| tomario-{env}-ecr-dkr-vpce | ECR DKR | Interface | あり | あり | あり（リリース後は常時） |
| tomario-{env}-s3-vpce | S3 | Gateway（無料） | あり | あり | あり（リリース後は常時） |
| tomario-{env}-logs-vpce | CloudWatch Logs | Interface | あり | あり | あり（リリース後は常時） |
| tomario-{env}-secretsmanager-vpce | Secrets Manager | Interface | あり | あり | あり（リリース後は常時） |
| tomario-{env}-ssmmessages-vpce | SSM Messages | Interface | あり | あり | あり（リリース後は常時。ECS Execによる障害対応・DBメンテナンス接続用） |

**2026-07-15追加：** `ssmmessages`エンドポイントはECS Exec（`aws ecs execute-command`、RDSバックアップ訓練・テストデータ投入時にコンテナへ接続するために使用）のセッション確立に必要。NAT Gatewayを使わない設計のため、このエンドポイントが無いとコンテナからSSMサービスへ到達できず接続が確立しない（`TargetNotConnectedException`）。既存の`VPCエンドポイント-SG`をそのまま使用しており、SG側の追加変更は不要。dev/stagingでは他の4つのインターフェースエンドポイントと同様cost-stop対象（`cost-stop.yml`のdestroy対象に追加済み）。

## 通信フロー

```
インターネット
    │
CloudFront（HTTPS終端・S3/ALBへのパス振り分け）
    │  /api/* のみALBへ（X-Origin-Verifyヘッダー付与）
Internet Gateway
    │
   ALB（パブリックサブネット）
    │
   ECSタスク（プライベートサブネット）─── VPCエンドポイント ─── ECR / CloudWatch Logs / Secrets Manager / SSM
    │
   RDS（プライベートサブネット）
```

---

## TBD解消事項

なし
