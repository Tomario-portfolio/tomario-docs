# セキュリティ

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
CloudFront（HTTPS終端、X-Origin-Verifyヘッダーを付与）
    │
┌──────────────────────┐
│   ALB-SG             │  HTTP（80）のみ許可。リスナールールでX-Origin-Verifyヘッダーを検証
└──────────────────────┘
    │
┌──────────────────────┐
│   ECS-SG             │  ALB-SGからのアプリケーションポートのみ許可
└──────────────────────┘
    │         │
    │    ┌──────────────────────┐
    │    │ VPCエンドポイント-SG    │  ECS-SGからのHTTPS（443）のみ許可
    │    └──────────────────────┘
    │
┌──────────────────────┐
│   RDS-SG             │  ECS-SGからのMySQL（3306）のみ許可
└──────────────────────┘
```

ALBはCloudFront経由のリクエストのみを処理する。CloudFrontがオリジンリクエストに付与するシークレットヘッダー（`X-Origin-Verify`、値はTerraformの`random_password`で生成）をALBのリスナールールで検証し、ALBへの直接アクセスを拒否する。

ポート番号・プロトコルの詳細は[環境定義書](../environment-definitions/)に記載する。

## IAM設計

### ECS Task Execution Role

ECSタスクの起動に必要な権限を付与するロール。ECSサービスが使用する。

| 権限 | 用途 |
|------|------|
| ECRからのイメージ取得 | コンテナイメージのプル |
| CloudWatch Logsへの書き込み | アプリケーションログの送信 |
| Secrets Managerからの値取得 | DB接続情報・SECRET_KEYの取得 |

### ECS Task Role

ECSタスク（アプリケーション）が使用するロール。
ECS Exec（障害対応・DBメンテナンス時のコンテナ接続）に必要な`ssmmessages`系の権限のみを付与し、それ以外は必要に応じて追加する。

### GitHub Actions

GitHub ActionsからAWSへの認証にはOIDCを使用し、アクセスキーを発行しない。
特定のGitHubリポジトリ・ブランチからのみ認証を許可する条件を設定する。
nonprod・prodはアカウントごとに別ロールとし、インフラ用（`tomario-infra`）とアプリデプロイ用（`tomario-app`）で権限を分離する。

## 機密情報管理

機密情報はコードやリポジトリに含めない。
以下の情報をSecrets Managerで管理し、ECSタスク起動時に環境変数として自動注入する。

| 機密情報 | 管理方法 |
|---------|---------|
| DB認証情報（ユーザー名・パスワード） | Secrets Manager（RDS管理、`manage_master_user_password`） |
| Flask SECRET_KEY | Secrets Manager（90日ごとに自動ローテーション） |

GitHub ActionsのIAMロールARNはGitHubリポジトリのSecretsで管理する。

SECRET_KEY（セッション署名用）は、Secrets Managerの自動ローテーション（ローテーション用Lambda）で90日ごとに新しいランダム値へ入れ替える。ローテーション後はECSタスクを入れ替えて新しい鍵を読み込ませ、1つ前の鍵もアプリに渡して受け付けることで、ローテーションしてもログイン中のセッションが切れないようにする。

## 検出・監視

### CloudTrail・GuardDuty（アカウント単位）

CloudTrail・GuardDutyはアカウント単位で管理するシングルトンリソースのため、workload環境ごとではなくアカウントごとに1つ作成する。

| アカウント | 管理場所 |
|----------|---------|
| nonprod | `envs/nonprod/shared`（dev/stagingで共有） |
| prod | `envs/prod/production/security`（workload環境がproductionのみのため、shared相当のディレクトリは作らない） |

**CloudTrail**：AWSアカウント上の全API操作を記録し、ログ集約用S3バケットに保存する。管理イベントのみを対象とし、データイベントは対象外とする。

**GuardDuty**：VPC Flow Logs・CloudTrail・DNSログを分析し、AWSアカウントへの脅威を継続的に検出する。常時有効とする。

### VPC Flow Logs

VPC内のネットワークトラフィック（送信元・宛先IP・ポート・プロトコル・許可/拒否）をS3（ログ集約バケット）へ記録する。
不正アクセスの痕跡調査やセキュリティグループルールの検証に使用する。

### セキュリティスタック（WAF・AWS Config・Security Hub、productionのみ）

WAF・AWS Config・Security Hubはproductionでのみ導入し、1つのフラグでまとめてON/OFFして必要な期間だけ有効化する（`tomario-infra`の`security-stack.yml`、cost-stop/startとは独立）。nonprodには導入しない。判断の経緯は[ADR: セキュリティスタックの「使う時だけ有効化」運用](../../tomario-steering/adr/infra/security/001-security-stack-cost-management.md)を参照。

| サービス | 内容 |
|---------|------|
| WAF | CloudFront用・ALB用のWeb ACLを作成し、AWS Managed Rule Groupsで既知のWebアプリケーション層の攻撃を防御する。ブロックしたリクエストはCloudWatch Logsへ出力 |
| AWS Config | 全リソースの設定変更履歴を記録（Security Hubの前提条件） |
| Security Hub | CIS AWS Foundations Benchmarkへの準拠状況を可視化 |

適用するルールグループ・CIS標準のバージョンは[環境定義書](../environment-definitions/security-environment-design.md)を参照。

### アカウントレベルの保険設定

未使用のサービスについても誤ってパブリック共有されないよう、以下をアカウントレベルで有効化している。

- EBSスナップショットのパブリックアクセスブロック
- SSM Documentsのパブリック共有ブロック
