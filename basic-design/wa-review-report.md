# Well-Architected レビューレポート

## レビュー概要

| 項目 | 内容 |
|---|---|
| レビュー対象 | Tomario ホテル予約システム — 基本設計書（high-level-specs 全10ファイル） |
| レビュー日 | 2026-06-30 |
| レビュアー | AWS Solutions Architect (Claude) |

---

## エグゼクティブサマリー

TomarioはECS Fargate・RDS MySQL・CloudFront・GitHub Actions OIDCを組み合わせたポートフォリオ向けホテル予約システムである。  
IaC（Terraform）による全リソース管理、OIDCによるアクセスキーレス認証、Secrets Managerでの機密情報管理、VPCエンドポイントによるNAT Gateway不使用設計など、ポートフォリオとして非常に高い設計品質を示している。  

重大度「高」の検出事項はなく、全体的に堅実な設計である。  
主な改善ポイントはセキュリティの検出コントロール（CloudTrail・GuardDuty）の未設定と、コスト可視化（AWS Budgets）の欠如であり、どちらも追加コストを最小限に抑えながら対応可能である。

### 柱別評価サマリー

| 柱 | 評価 | 重大 | 高 | 中 | 低 |
|---|---|---|---|---|---|
| 運用上の優秀性 | 良好 | 0 | 0 | 1 | 3 |
| セキュリティ | 改善余地あり | 0 | 0 | 3 | 3 |
| 信頼性 | 良好 | 0 | 0 | 2 | 2 |
| パフォーマンス効率 | 良好 | 0 | 0 | 0 | 4 |
| コスト最適化 | 改善余地あり | 0 | 0 | 2 | 2 |
| サステナビリティ | 良好 | 0 | 0 | 0 | 3 |

---

## 柱別レビュー結果

### 運用上の優秀性

#### 良好な点

- Terraformで全リソースをコード管理しており、環境の完全再現性が保証されている。手動操作が排除されている設計は高く評価できる。
- GitHub Actions でインフラCIとアプリデプロイの両方を自動化しており、ヒューマンエラーのリスクが低い。
- mainブランチ保護（PR必須・ステータスチェック必須）により、直接pushによる構成変更が防止されている。
- ECSタスクのアプリケーションログをCloudWatch Logsに収集し（保持期間7日）、障害時の調査基盤が整っている。
- CloudWatch AlarmとSNSの組み合わせでALB異常ホスト・ECS CPU・RDS CPUの3種のアラームが設定されている。

#### 検出事項

| # | 重要度 | 検出事項 | 推奨事項 | 関連サービス |
|---|---|---|---|---|
| OPS-1 | 中 | ロールバック手順が設計書に未記載。デプロイ失敗時の復旧手順が不明確 | 旧イメージタグを指定したECSタスク定義の更新手順をRunbookとして記載する | ECS, ECR |
| OPS-2 | 低 | カスタムメトリクスが未収集。予約件数・APIレスポンスタイム等のアプリレベル指標がない | 将来的にFlaskのメトリクスをCloudWatch EMF（Embedded Metrics Format）で送信する構成を検討する | CloudWatch |
| OPS-3 | 低 | Blue/Green・Canaryなどの段階的リリース戦略が未採用。ローリングアップデートのみ | dev環境では現構成で十分だが、prd向けにはCodeDeploy + ECS Blue/Greenデプロイの採用を検討する | CodeDeploy, ECS |
| OPS-4 | 低 | stg環境が未設計。devからprdへの直接リリースが想定されており、prd相当の事前検証が困難 | 要件定義書に「将来対応」と記載済み。prd設計時にstgも並行して計画することを推奨する | — |

#### 補足説明

OPS-1のロールバックについて、ECSはIMMUTABLEタグを採用しているため旧イメージが保存されている（最新5世代のライフサイクルポリシー）。タスク定義で旧イメージタグを指定してECSサービスを更新するだけでロールバック可能であり、手順自体はシンプルである。設計書への記載を優先事項としたい。

---

### セキュリティ

#### 良好な点

- GitHub ActionsのOIDC認証によりアクセスキーを完全に排除している。特定リポジトリ・ブランチからのみ認証を許可する条件設定も優れた実装である。
- DB認証情報・Flask SECRET_KEYの両方をSecrets Managerで管理し、ECSタスク起動時に自動注入している。コードへのハードコードが完全に排除されている。
- SG連鎖（インターネット→ALB→ECS→RDS/VPCエンドポイント）による多層防御が正しく実装されており、各レイヤーで最小権限のネットワークアクセスが確保されている。
- ECSのTaskExecutionRoleとTaskRoleが分離されており、最小権限の原則が適用されている。
- ECRのIMMUTABLEタグとプッシュ時脆弱性スキャンにより、イメージの不変性と脆弱性の早期検出が実現されている。
- CloudFrontがHTTPS終端を担い、ALBはCloudFront経由のリクエストのみ受け付ける構成になっている。
- RDSのストレージ暗号化（`storage_encrypted = true`）が有効化されている。

#### 検出事項

| # | 重要度 | 検出事項 | 推奨事項 | 関連サービス |
|---|---|---|---|---|
| SEC-1 | 中 | CloudTrailが設計書・Terraformコードに言及なし。AWS API操作の監査ログが存在しない | CloudTrailを有効化する。管理イベントのみであれば最初の10万件/月は無料。S3バケットへの証跡保存を設定する | CloudTrail, S3 |
| SEC-2 | 中 | GuardDutyが未設定。異常な通信・認証情報の不正使用等の脅威が自動検出されない | GuardDutyを有効化する。30日間の無料試用期間がある。要件定義書に「prod向け将来対応」と明記されているが、devでも有効化を推奨する | GuardDuty |
| SEC-3 | 中 | VPC Flow Logsが未設定。不審なネットワーク通信の検知が困難 | VPC Flow LogsをCloudWatch Logsへ送信する設定を追加する。保持期間は7日程度に設定してコストを抑える | VPC, CloudWatch Logs |
| SEC-4 | 低 | アプリケーション依存ライブラリの脆弱性チェックがCI/CDに未組み込み | GitHub ActionsにDependabotアラートまたは`pip-audit`ステップを追加する | GitHub Actions |
| SEC-5 | 低 | AWS WAFが未設定。SQLインジェクション・XSSなどのアプリケーション層攻撃への防御がない | 要件定義書に「prod向け将来対応」と明記済み。prd設計時にAWS WAF（CloudFront + ALBへの適用）を計画に含める | WAF, CloudFront, ALB |
| SEC-6 | 低 | Security Hubが未設定。セキュリティ検出結果が一元管理されていない | GuardDuty・CloudTrail有効化後にSecurity Hubを有効化すると検出結果が集約される。30日無料試用あり | Security Hub |

#### 補足説明

SEC-1（CloudTrail）はセキュリティインシデント発生時の調査に不可欠であり、ポートフォリオでも有効化することを強く推奨する。Terraformでは以下のように1リソースで追加できる。

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "tomario-${var.env}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = false
}
```

管理イベントの記録のみであれば追加コストは事実上ゼロである（データイベントは有料）。

---

### 信頼性

#### 良好な点

- Multi-AZ対応のVPC・サブネット設計（ap-northeast-1a/1c の2AZ）が実装されており、prd環境でのMulti-AZ展開に即対応できる基盤が整っている。
- ALBのヘルスチェックとECSサービスの組み合わせにより、異常タスクへのルーティング停止と自動タスク再起動が実現されている。
- RDS自動バックアップ（保持7日間）とポイントインタイムリカバリ（RPO: 最大5分）が設定されており、データ保護が適切に設計されている。
- RTO・RPOが要件定義書・設計書ともに定義されている（RPO: 最大5分、RTO: 約1時間）。
- VPCエンドポイントによりECR・CloudWatch Logs・Secrets ManagerへのAWS内ネットワーク接続が保証されている。

#### 検出事項

| # | 重要度 | 検出事項 | 推奨事項 | 関連サービス |
|---|---|---|---|---|
| REL-1 | 中 | dev環境のECSタスク数が1のみで、タスク障害時に手動復旧が必要になるまでの間サービスが停止する | ECSサービスの自動タスク再起動は機能しているが、再起動までの数十秒間はダウンタイムが発生する。dev環境の許容範囲として設計書に明記する | ECS |
| REL-2 | 中 | ECS Auto Scalingが未設定。prd環境でトラフィック増加時にタスク数が増えない | Application Auto Scalingを定義し、CPU使用率70%以上でタスク数を増加させるポリシーをprd設計に含める | ECS, Application Auto Scaling |
| REL-3 | 低 | RDSバックアップのリストアテストが計画されていない | 半年に1回程度、スナップショットからリストアして動作確認する手順を設計書に記載する | RDS |
| REL-4 | 低 | RDS停止後7日でAWSが自動起動する仕様への対策が運用レベルにとどまっている | Lambda + EventBridgeで定期的にRDSを自動停止するスクリプトを追加することで運用負担を軽減できる | RDS, Lambda, EventBridge |

#### 補足説明

REL-4のRDS自動起動問題は、コスト設計書に「毎週手動で停止し直す運用とする」と正直に記載されている点は評価できる。ただしポートフォリオとして将来的には自動化を示すことで、運用設計力のアピールになる。

---

### パフォーマンス効率

#### 良好な点

- ECS FargateによるサーバーレスコンピューティングでOSレベルの管理が不要になり、リソース効率が最適化されている。
- RDSにgp3ストレージを採用しており、gp2と比較して同コストでより高いIOPS・スループットが得られる。
- CloudFrontによるCDNエッジキャッシュで静的コンテンツのレイテンシーが削減されており、S3とALBへのオリジンアクセスを一元管理している。
- ALBによる2AZへのトラフィック分散で、ECSタスクへの負荷が均等化されている。

#### 検出事項

| # | 重要度 | 検出事項 | 推奨事項 | 関連サービス |
|---|---|---|---|---|
| PERF-1 | 低 | ECSタスクのCPU（0.25vCPU）・メモリ（0.5GB）サイズの妥当性が未検証 | CloudWatchのECS CPUメトリクスを数週間観測し、実際の使用率に基づいてサイズを検証する | CloudWatch, ECS |
| PERF-2 | 低 | Graviton（ARM64）ベースのFargateタスクが未検討。同等スペックでコストが約20%低減できる | Dockerfileにマルチプラットフォームビルドを追加し、`--platform linux/arm64`でビルドすることで対応可能 | ECS Fargate |
| PERF-3 | 低 | 負荷テストが未実施。ピーク時のレスポンスタイム特性が不明 | Apache Bench（ab）またはLocustを使用した簡易負荷テストをdev環境で実施し、設計書にその結果を記録する | — |
| PERF-4 | 低 | RDS Proxyが未検討。ECSタスクが複数になった場合のDB接続数管理が課題になりうる | prd・スケールアウト設計時にRDS Proxyの導入を検討する | RDS Proxy |

---

### コスト最適化

#### 良好な点

- cost-stop/startワークフローにより、ALB・VPCエンドポイント・ECSタスクを不使用時に停止・削除するパターンが正しく設計されている。月額コストを~$55から~$1に削減できており、dev環境のコスト最適化として非常に優れた実装である。
- NAT Gatewayを使用せずVPCエンドポイントで代替しており、月~$32のNAT Gatewayコストをゼロにしている。
- ECRライフサイクルポリシーで最新5世代のみ保持しており、不要なイメージによるストレージコストの増大を防いでいる。
- Secrets Manager 2件（DB認証情報・Flask SECRET_KEY）のコストが正確に設計書に反映されている（~$0.80/月）。

#### 検出事項

| # | 重要度 | 検出事項 | 推奨事項 | 関連サービス |
|---|---|---|---|---|
| COST-1 | 中 | AWS Budgetsが未設定。コスト超過に請求書が届くまで気付けない | 月$10程度の予算アラートをAWS Budgetsで設定する。Terraformで`aws_budgets_budget`リソースとして管理できる | AWS Budgets |
| COST-2 | 中 | Cost Anomaly Detectionが未設定。異常なコストスパイクの早期検知ができない | Cost Anomaly Detectionモニターを作成し、SNSへの通知を設定する。最初の$100の検知は追加料金なし | Cost Anomaly Detection, SNS |
| COST-3 | 低 | コスト配分タグの設計が未記載。NameタグはあるがEnvironment/Projectタグの設計方針がない | 全リソースに`Project = tomario`・`Environment = dev/prd`タグを付与するポリシーをコスト設計書に明記し、Terraformのデフォルトタグとして設定する | Cost Explorer |
| COST-4 | 低 | RDS停止後7日でAWSが自動起動する問題が手動運用に依存している | Lambda + EventBridgeによる定期自動停止で解決できる | RDS, Lambda, EventBridge |

#### 補足説明

COST-3のデフォルトタグは、Terraformのproviderブロックに以下を追記するだけで全リソースに自動付与できる。

```hcl
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Project     = "tomario"
      Environment = var.env
      ManagedBy   = "terraform"
    }
  }
}
```

---

### サステナビリティ

#### 良好な点

- ECS FargateはコンテナネイティブなマネージドサービスでありEC2と異なりアイドル時のリソース占有がなく、電力効率が高い。
- cost-stop/startにより開発環境の不使用時にALB・VPCエンドポイント・ECSタスクが停止され、無駄なリソース稼働が排除されている。
- ECRライフサイクルポリシーとCloudWatch Logsの保持期間設定（7日）により、不要データの蓄積が抑制されている。
- CloudFrontのエッジキャッシュにより、静的コンテンツに対するオリジンへの不要なリクエストが削減されている。

#### 検出事項

| # | 重要度 | 検出事項 | 推奨事項 | 関連サービス |
|---|---|---|---|---|
| SUS-1 | 低 | Graviton（ARM64）ベースのFargateが未検討。同等性能でCPUあたりの電力消費を約60%削減できる | PERF-2と同じ対応。`runtimePlatform`に`cpuArchitecture = ARM64`を指定する | ECS Fargate |
| SUS-2 | 低 | S3フロントエンドバケットのライフサイクルポリシーが設計書に未記載 | 静的ファイルは頻繁には変わらないため、90日後にS3 Intelligent-Tieringに移行するポリシーを検討する | S3 |
| SUS-3 | 低 | RDS 7日自動起動により停止中にリソースが意図せず稼働する | REL-4・COST-4と同じ対応（Lambda自動停止）で解決できる | RDS, Lambda |

---

## 横断的な推奨事項

**CloudTrail・GuardDuty・AWS Budgetsの3点セット**が最優先の横断的改善項目である。  
これら3つは合計で月額数百円程度（いずれも無料枠・試用期間あり）で導入可能であり、セキュリティと運用の基盤として不可欠である。  
ポートフォリオとして「検出・監視・コスト管理まで設計した」ことを示せるため、評価上のインパクトも大きい。

**Lambda + EventBridgeによるRDS定期自動停止**は、信頼性・コスト・サステナビリティの3つの柱にまたがる改善効果がある。  
毎週手動で停止する運用をコード化することで、運用自動化の設計力をポートフォリオで示すことができる。

---

## 次のアクション（優先順位順）

| 優先度 | アクション | 関連する柱 | 期待される効果 |
|---|---|---|---|
| 1 | CloudTrailを有効化してS3に証跡を保存するTerraformリソースを追加する | セキュリティ | API操作の監査ログを確保。追加コストほぼゼロ |
| 2 | AWS Budgets（月$10アラート）をTerraformで定義する | コスト最適化 | コスト超過の早期検知。月額無料 |
| 3 | GuardDutyを有効化する（30日無料試用） | セキュリティ | 不正アクセス・認証情報漏洩の自動検出 |
| 4 | デプロイロールバック手順をRunbookとして設計書に追記する | 運用上の優秀性 | 障害対応の手順を明文化。実装コストゼロ |
| 5 | Lambda + EventBridgeによるRDS定期自動停止を実装する | 信頼性・コスト・サステナビリティ | 手動停止運用をコード化。ポートフォリオ価値向上 |

---

## 参考リンク

- [AWS Well-Architected フレームワーク](https://docs.aws.amazon.com/ja_jp/wellarchitected/latest/framework/welcome.html)
- [AWS CloudTrail — 開始方法](https://docs.aws.amazon.com/ja_jp/awscloudtrail/latest/userguide/cloudtrail-getting-started.html)
- [Amazon GuardDuty — 開始方法](https://docs.aws.amazon.com/ja_jp/guardduty/latest/ug/getting_started.html)
- [AWS Budgets — 予算の作成](https://docs.aws.amazon.com/ja_jp/cost-management/latest/userguide/budgets-create.html)
- [ECS Fargate Graviton — ARM64対応](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/ecs-graviton.html)
- [Amazon EventBridge + Lambda — スケジュールイベント](https://docs.aws.amazon.com/ja_jp/eventbridge/latest/userguide/eb-run-lambda-schedule.html)
