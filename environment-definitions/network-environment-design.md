# network 環境定義

VPC・サブネット・IGW・ルートテーブル・VPCエンドポイント・VPC Flow Logsを管理するコンポーネント（`modules/network`）。
workload環境（dev/staging/production）ごとに独立したVPCを作成する。`shared`はVPCを持たない。

---

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

全環境とも同じ構成で、CIDRの第2オクテットのみ環境ごとに異なる（dev=10.0、staging=10.1、production=10.2）。

| サブネット名 | 種別 | AZ | CIDR（dev / staging / production） | 配置リソース | パブリックIP自動割り当て |
|------------|------|----|------|------------|-------------------|
| tomario-{env}-public-ap-northeast-1a | パブリック | ap-northeast-1a | 10.0.0.0/24 / 10.1.0.0/24 / 10.2.0.0/24 | ALB | 有効 |
| tomario-{env}-public-ap-northeast-1c | パブリック | ap-northeast-1c | 10.0.1.0/24 / 10.1.1.0/24 / 10.2.1.0/24 | ALB | 有効 |
| tomario-{env}-private-ap-northeast-1a | プライベート | ap-northeast-1a | 10.0.10.0/24 / 10.1.10.0/24 / 10.2.10.0/24 | ECSタスク・RDS | 無効 |
| tomario-{env}-private-ap-northeast-1c | プライベート | ap-northeast-1c | 10.0.11.0/24 / 10.1.11.0/24 / 10.2.11.0/24 | ECSタスク・RDS | 無効 |

## Internet Gateway

| 項目 | dev | staging | production |
|------|-----|---------|---|
| IGW名 | tomario-dev-igw | tomario-staging-igw | tomario-production-igw |

## ルートテーブル

| ルートテーブル名 | 関連サブネット | ルート |
|--------------|-------------|--------|
| tomario-{env}-public-rt | パブリックサブネット×2 | 0.0.0.0/0 → IGW |
| tomario-{env}-private-rt | プライベートサブネット×2 | VPC内通信のみ（local）＋S3 Gatewayエンドポイント |

## NAT Gateway

全環境で使用しない（VPCエンドポイントで代替）。

---

## VPC Flow Logs

| 項目 | dev | staging | production |
|------|-----|---------|---|
| リソース名 | tomario-dev-flow-log | tomario-staging-flow-log | tomario-production-flow-log |
| トラフィック種別 | ALL（許可・拒否を両方記録） | ALL | ALL |
| 送信先 | S3ログ集約バケット（[logging-environment-design.md](logging-environment-design.md)参照） | 同左 | 同左 |
| 出力先パス | `/flow-logs/AWSLogs/{account_id}/*` | 同左 | 同左 |
| 保持期間 | 7日 | 30日 | 90日 |
| IAMロール | 不要（S3配信はバケットポリシーで許可） | 同左 | 同左 |

---

## VPCエンドポイント

Interface型はcost-stopの対象（未使用時は削除）。productionは一般公開前は同じ扱いとし、公開後は常時作成しておく（NAT Gatewayが無いため、エンドポイントが無いとECSタスクがAWSサービスへ到達できない）。

### VPCエンドポイント-SG

| 項目 | dev | staging | production |
|------|-----|---------|---|
| SG名 | tomario-dev-vpce-sg | tomario-staging-vpce-sg | tomario-production-vpce-sg |

| 方向 | プロトコル | ポート | 送信元 |
|------|----------|-------|--------|
| インバウンド | TCP | 443 | ECS-SG |
| アウトバウンド | 全て | 全て | 0.0.0.0/0 |

### エンドポイント一覧

全環境で同じ構成とする。

| エンドポイント名 | サービス | 種別 | cost-stop対象 | 用途 |
|--------------|---------|------|-------------|------|
| tomario-{env}-ecr-api-vpce | ECR API | Interface | 対象 | イメージ取得（API操作） |
| tomario-{env}-ecr-dkr-vpce | ECR DKR | Interface | 対象 | イメージ取得（Docker通信） |
| tomario-{env}-s3-vpce | S3 | Gateway（無料） | 対象外 | ECRイメージレイヤーの取得 |
| tomario-{env}-logs-vpce | CloudWatch Logs | Interface | 対象 | ログ送信 |
| tomario-{env}-secretsmanager-vpce | Secrets Manager | Interface | 対象 | DB接続情報・SECRET_KEYの取得 |
| tomario-{env}-ssmmessages-vpce | SSM Messages | Interface | 対象 | ECS Execのセッション確立（無いと`TargetNotConnectedException`になる） |

---

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

## 未解決事項

なし
