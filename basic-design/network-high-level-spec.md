# ネットワーク

## リージョン・環境

東京リージョン（ap-northeast-1）を使用する。
環境は nonprod（dev / staging）／ prod（production）とし、workload環境ごとに独立したVPCを作成する。
`shared`（nonprodアカウント共通のGuardDuty・CloudTrail・Budgets・ECR等）はアカウント単位のリソースのみでVPCを持たない。

将来VPCピアリングが必要になるケースに備え、環境間でCIDRが重複しない設計とする。

| 環境 | CIDR |
|------|------|
| dev | 10.0.0.0/16 |
| staging | 10.1.0.0/16 |
| production | 10.2.0.0/16（別アカウントのため厳密な重複回避は必須ではないが、将来のVPCピアリング等に備えて分けている） |

## Availability Zone

可用性確保のため、ap-northeast-1a と ap-northeast-1c の2AZ構成とする。
1AZで障害が発生した場合も、残りのAZでサービスを継続できる。

## VPC

環境ごとに1つのVPCを作成する。
VPC内のリソースがAWSのDNSサーバーを利用できるよう、DNSサポートおよびDNSホスト名を有効にする。

## サブネット設計

VPC内はルートテーブルを基準にパブリックサブネットとプライベートサブネットに分離する。

| サブネット種別 | ルートテーブル | 配置リソース |
|-------------|-------------|------------|
| パブリックサブネット | Internet Gatewayへの経路を持つ | ALB |
| プライベートサブネット | VPC内部通信に限定 | ECSタスク・RDS |

ALBのみをパブリックサブネットに配置し、ECSタスク・RDSはプライベートサブネットに配置する。
インターネットに直接公開するリソースをALBのみに限定することでセキュリティを強化する。

## Internet Gateway

インターネットアクセスが必要なリソース（ALB）が存在するため、Internet Gatewayを使用する。

## NAT Gateway

NAT Gatewayは使用しない（全環境共通の方針）。
ECSタスクからAWSサービスへのアクセスはVPCエンドポイント経由で行う。

NAT Gatewayが無いことに起因する到達性の問題が過去に一度発生し、`bootstrap_image`の参照先変更で解決した。経緯は[ADR: bootstrap_imageのプライベートECR参照化](../../tomario-steering/adr/infra/network/002-bootstrap-image-private-ecr.md)を参照。

## VPCエンドポイント

ECSタスク（プライベートサブネット）がAWSサービスに接続するためVPCエンドポイントを使用する。
インターネットを経由せずAWS内部ネットワークで接続するため、NAT Gatewayより安全。
Interface型は時間課金のため、全環境で作業時以外は削除する運用とする（cost-stop対象）。

| エンドポイント | 種別 | 用途 |
|-------------|------|------|
| ECR API | Interface | ECSタスクのイメージ取得（API操作） |
| ECR DKR | Interface | ECSタスクのイメージ取得（Docker通信） |
| S3 | Gateway | ECRイメージレイヤーの取得（無料） |
| CloudWatch Logs | Interface | ECSタスクのログ送信 |
| Secrets Manager | Interface | DB接続情報・SECRET_KEYの取得 |
| SSM Messages | Interface | ECS Exec（障害対応・DBメンテナンス時のコンテナ接続）のセッション確立 |

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

## VPC Flow Logs

VPC内の全ネットワークトラフィックをS3（ログ集約バケット）に記録する。
セキュリティインシデント時の通信経路の追跡やセキュリティグループルールの検証に使用する。
保持期間は環境ごとのログ保持方針（dev=7日／staging=30日／production=90日、詳細は[monitoring-high-level-spec.md](monitoring-high-level-spec.md)参照）に準ずる。

## 命名規則

リソース名は `{システム名}-{環境}-{リソース}` の形式で統一する（詳細は[naming-high-level-spec.md](naming-high-level-spec.md)参照）。
障害対応時にリソースの所属環境を即座に識別できるようにするため。
