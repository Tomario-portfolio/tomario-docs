# セキュリティ

## 要件

## 基本方針

最小権限の原則に基づき、各リソースに必要最低限のアクセスのみを許可する。
セキュリティグループで通信経路を制限し、機密情報はコードに直接記載せずSecrets Managerで管理する。
AWS認証はOIDCを使用しアクセスキーを発行しない。

## ネットワークセキュリティ

各リソース間のアクセス制限にはセキュリティグループを使用し、必要な通信のみを許可する。
ALB・ECSタスク・RDS・VPCエンドポイントそれぞれにセキュリティグループを設け、上位リソースからのみ通信を許可する構成とする。

```
インターネット
    │
┌─────────────┐
│   ALB-SG    │  インターネットからのHTTP/HTTPSのみ許可
└─────────────┘
    │
┌─────────────┐
│  ECS-SG     │  ALBからのみ許可
└─────────────┘
    │         │
    │    ┌─────────────────┐
    │    │  VPCエンドポイント-SG │  ECS-SGからのHTTPS（443）のみ許可
    │    └─────────────────┘
    │
┌─────────────┐
│   RDS-SG    │  ECSからのMySQL接続のみ許可
└─────────────┘
```

<!--
## ネットワークセキュリティ（旧）

┌─────────────┐
│   ALB-SG    │  インターネットからのHTTP/HTTPSのみ許可
└─────────────┘
    │
┌─────────────┐
│   EC2-SG    │  ALBからのみ許可
└─────────────┘
    │
┌─────────────┐
│   RDS-SG    │  EC2からのMySQL接続のみ許可
└─────────────┘
-->

ポート番号・プロトコルの詳細は詳細設計書に記載する。

## IAM設計

### ECS Task Execution Role

ECSタスクの起動に必要な権限を付与するロール。ECSサービスが使用する。

| 権限 | 用途 |
|------|------|
| ECRからのイメージ取得 | コンテナイメージのプル |
| CloudWatch Logsへの書き込み | アプリケーションログの送信 |
| Secrets Managerからの値取得 | DB接続情報の取得 |

### ECS Task Role

ECSタスク（アプリケーション）が使用するロール。アプリケーションが直接AWSサービスを呼び出す場合に付与する。
現時点では最小権限とし、必要に応じて追加する。

<!--
## IAM設計（旧）

EC2インスタンスにはIAMロールを付与し、SSM接続とSecrets ManagerからのDB接続情報取得のみを許可する。
EC2に不必要に広い権限を与えず、侵害時の被害範囲を最小化する。
-->

GitHub ActionsからAWSへの認証にはOIDCを使用する。
アクセスキーを発行しないことでキー漏洩リスクをゼロにする。
特定のGitHubリポジトリ・ブランチからのみ認証を許可する条件を設定する。

## 機密情報管理

機密情報はコードやリポジトリに含めない。
以下の情報をSecrets Managerで管理し、ECSタスク起動時に環境変数として自動注入する。

| 機密情報 | 管理方法 |
|---------|---------|
| DB認証情報（ユーザー名・パスワード） | Secrets Manager（RDS管理、自動ローテーション有効） |
| Flask SECRET_KEY | Secrets Manager（手動登録、**ローテーション未設定**） |

GitHub ActionsのIAMロールARNはGitHubリポジトリのSecretsで管理する。

### 既知の改善余地：Flask SECRET_KEYのローテーション（2026-07-11、ベストプラクティスレビューで発見）

DB認証情報はRDS管理のSecrets Managerにより自動ローテーションされるが、`SECRET_KEY`（セッション署名用）は手動登録のみでローテーション方針が無い。長期間同じ鍵を使い続けることになるため、将来的にSecrets Managerのカスタムローテーション用Lambdaを追加し、定期的なローテーションを実施することを検討する（`todo.md`に記録）。

## 検出・監視

### CloudTrail・GuardDuty・Budgets（アカウント単位、shared環境に集約）

CloudTrail・GuardDuty・AWS Budgets・Cost Anomaly Detectionは、いずれもアカウント/リージョンにつき1つまでしか作成できない（またはアカウント単位で管理するのが自然な）シングルトンリソースである。
workload環境（dev/staging/production）ごとに重複して作成すると、2つ目以降がAPIエラーになる、または意味が無い。
そのため`envs/nonprod/shared`という専用ディレクトリ・専用stateに切り出し、nonprodアカウント共通のリソースとして一元管理する（2026-07-10、`terraform state mv`による無停止移行で実施）。

**CloudTrail**：AWSアカウント上の全API操作を記録し、S3バケットに保存する。管理イベントのみを対象とし、データイベントは対象外とする（コスト抑制）。保存先は`modules/logging`が管理するログ集約用S3バケットに統合済み（2026-07-13、既存バケットを`terraform state mv`で無停止移行）。

**GuardDuty**：AWSアカウントへの脅威を継続的に検出するマネージドサービス。VPC Flow Logs・CloudTrail・DNSログを機械学習で分析し、不審な操作を自動検知する。

**Budgets / Cost Anomaly Detection**：予算超過アラート・異常課金検知。

production環境は別AWSアカウント（`tomario-prod-sysadmin`、2026-07-25作成済み、bootstrap-prod構築済み）のため、CloudTrail・GuardDuty・Budgets・Cost Anomaly Detectionはprodアカウント側で別途有効化する（nonprod側とは完全に独立）。
ただしnonprodと異なり、production環境はworkload環境がproductionの1つのみのため`shared`ディレクトリは作らず、`envs/prod/production`配下に直接同居させる。

### VPC Flow Logs

VPC内のネットワークトラフィック（送信元・宛先IP・ポート・プロトコル・許可/拒否）を記録する。
S3（`modules/logging`のログ集約バケット）へ直接出力し、保持期間はログ保持方針（dev=7日／staging=30日／production=90日予定）に準ずるライフサイクルポリシーで管理する（2026-07-13、元はCloudWatch Logs送信だったがS3直接出力に変更。プライベートサブネットのみのVPCでもIAMロール不要でシンプルになるため）。
不正アクセスの痕跡調査やセキュリティグループルールの検証に使用する。

### Security Hub / AWS Config（nonprodでは無効、production構築時に有効化予定）

CIS AWS Foundations Benchmarkへの準拠状況を可視化するためにSecurity Hubを使用する。
Security HubにはAWS Configの有効化が必要であり、Configはリソース変更のたびに記録コストが発生する。
非商用のnonprod環境（dev/staging/shared）でCIS準拠チェックまで行う必要性は薄いと判断し、nonprod/sharedでは無効のままとする。production環境では有効化する方針。

### WAF（2026-07-11：導入する方針）

CloudFront・ALB双方にWAF（AWS Managed Rule Groups）を導入する方針。
当初は非商用ポートフォリオのため費用対効果が薄いと見送っていたが、WAFのWeb ACLは「停止」ができず「存在（課金）／削除（無課金）」の二択かつ時間按分課金であることが判明。production環境はリリース後は常時稼働だが、WAF・Security Hub+AWS Configについては常時起動せず**面接が近いタイミングだけ**作成・有効化する運用とする（詳細な月額試算は[cost-high-level-spec.md](cost-high-level-spec.md)参照）。
実装（cost-stop.yml/cost-start.ymlへの組み込み含む）はproduction構築のタイミングで対応する（`todo.md`のSEC-5参照）。
