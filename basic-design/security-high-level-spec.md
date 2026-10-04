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
│   ECS-SG             │  ALB-SGからの8080のみ許可
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

GitHub ActionsからAWSへの認証にはOIDCを使用する。
アクセスキーを発行しないことでキー漏洩リスクをゼロにする。
特定のGitHubリポジトリ・ブランチからのみ認証を許可する条件を設定する。
nonprod・prodはアカウントごとに別ロールとし、インフラ用（`tomario-infra`）とアプリデプロイ用（`tomario-app`）で権限を分離する。

## 機密情報管理

機密情報はコードやリポジトリに含めない。
以下の情報をSecrets Managerで管理し、ECSタスク起動時に環境変数として自動注入する。

| 機密情報 | 管理方法 |
|---------|---------|
| DB認証情報（ユーザー名・パスワード） | Secrets Manager（RDS管理、`manage_master_user_password`） |
| Flask SECRET_KEY | Secrets Manager。productionはTerraformの`random_password`で生成した値を登録（**ローテーション未設定**） |

GitHub ActionsのIAMロールARNはGitHubリポジトリのSecretsで管理する。

### 既知の改善余地：Flask SECRET_KEYのローテーション

`SECRET_KEY`（セッション署名用）は初回登録のみでローテーション方針が無い。長期間同じ鍵を使い続けることになるため、将来的にSecrets Managerのカスタムローテーション用Lambdaを追加し、定期的なローテーションを実施することを検討する。

## 検出・監視

### CloudTrail・GuardDuty（アカウント単位）

CloudTrail・GuardDutyは、アカウント/リージョンにつき1つまでしか作成できない（またはアカウント単位で管理するのが自然な）シングルトンリソースである。
workload環境ごとに重複して作成すると、2つ目以降がAPIエラーになる、または意味が無い。

| アカウント | 管理場所 |
|----------|---------|
| nonprod | `envs/nonprod/shared`（dev/stagingで共有。2026-07-10、`terraform state mv`による無停止移行で集約） |
| prod | `envs/prod/production/security`（workload環境がproductionのみのため、shared相当のディレクトリは作らない） |

**CloudTrail**：AWSアカウント上の全API操作を記録し、ログ集約用S3バケット（`modules/logging`）に保存する。管理イベントのみを対象とし、データイベントは対象外とする（コスト抑制）。

**GuardDuty**：AWSアカウントへの脅威を継続的に検出するマネージドサービス。VPC Flow Logs・CloudTrail・DNSログを機械学習で分析し、不審な操作を自動検知する。停止できないサービスのため常時有効とする。

### VPC Flow Logs

VPC内のネットワークトラフィック（送信元・宛先IP・ポート・プロトコル・許可/拒否）を記録する。
S3（`modules/logging`のログ集約バケット）へ直接出力し、保持期間はログ保持方針（dev=7日／staging=30日／production=90日）に準ずるライフサイクルポリシーで管理する。
不正アクセスの痕跡調査やセキュリティグループルールの検証に使用する。

### セキュリティスタック（WAF・AWS Config・Security Hub、productionのみ）

WAF・AWS Config・Security Hubは、productionでのみ導入し、`enable_security_stack`フラグ1つでまとめてON/OFFして**必要な期間だけ有効化する**運用とする（`tomario-infra`の`security-stack.yml`、cost-stop/startとは独立）。判断の経緯は[ADR: セキュリティスタックの「使う時だけ有効化」運用](../../tomario-steering/adr/infra/security/001-security-stack-cost-management.md)を参照。

| サービス | 内容 |
|---------|------|
| WAF | CloudFront用（us-east-1）・ALB用のWeb ACLを作成。AWS Managed Rule Groups（`AWSManagedRulesCommonRuleSet`・`AWSManagedRulesKnownBadInputsRuleSet`・`AWSManagedRulesAmazonIpReputationList`）を適用。ブロックしたリクエストはCloudWatch Logsへ出力 |
| AWS Config | 全リソースの設定変更履歴を記録（Security Hubの前提条件） |
| Security Hub | CIS AWS Foundations Benchmark v1.4.0への準拠状況を可視化 |

nonprod（dev/staging/shared）は非商用の検証環境のため、CIS準拠チェックやWAFまで行う必要性は薄いと判断し、導入しない。

### アカウントレベルの保険設定

Security Hubの検出結果への対応として、未使用のサービスについても誤ってパブリック共有されないよう、以下をアカウントレベルで有効化している。

- EBSスナップショットのパブリックアクセスブロック
- SSM Documentsのパブリック共有ブロック
