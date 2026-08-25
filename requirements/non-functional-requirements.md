# 非機能要件定義書

## 1. 可用性

| 項目 | dev | staging | production（将来・方向性のみ） | 判断理由 |
|------|-----|---------|---|---|
| 稼働率SLA | 規定なし | 規定なし（検証目的） | 未定（常時稼働を想定） | dev/stagingはポートフォリオ・検証用途のため省略 |
| 運用時間 | cost-stopで夜間・休日停止 | 負荷テスト時のみ起動、cost-stop/startで管理 | 未定 | コスト削減を優先 |
| Multi-AZ | なし（単一AZインスタンス） | なし（変数化のみ。オン/オフは検討中） | 未定 | 非商用ポートフォリオでコスト（インスタンス料金がほぼ倍）に見合う必要性が薄いと判断。将来変更の可能性あり |
| Auto Scaling | 無効（desired_count=1固定） | 有効（desired_count=2、min=2/max=4、target CPU 70%） | staging構成を引き継ぐ想定 | stagingで水平スケーリングの実挙動を検証するために新規実装 |
| バックアップ | RDS自動バックアップ7日間保持 | 同左＋ポイントインタイムリストア訓練を実施 | 未定 | 本番で初めてリストアを試すリスクを避けるため、stagingで手順を検証 |
| RPO/RTO | 規定なし（ポートフォリオ用途） | 規定なし（検証目的） | 未定 | 現フェーズでは厳密な目標値は設定していない |

---

## 2. 性能・拡張性

| 項目 | dev | staging | production（将来・方向性のみ） | 判断理由 |
|------|-----|---------|---|---|
| APIレスポンス | 1秒以内（目標） | 同左 | 同左 | ポートフォリオ用途のため厳密な計測は省略 |
| ECSタスク数 | desired_count=1 | desired_count=2、Auto Scaling有効（min=2/max=4、target CPU 70%） | staging構成を引き継ぐ想定 | 水平スケーリングの検証が目的 |
| ECSリソース（CPU/メモリ） | CPU:256 / Memory:512 | **CPU:256 / Memory:512（変更なし）** | 未定 | 垂直スケールはせず、水平（タスク数）のみ増やす方針（2026-07-11決定） |
| DBインスタンスクラス | db.t3.micro | 通常時：db.t3.micro／負荷テスト直前のみAWS CLIで**db.t4g.medium**に一時変更 | 未定 | Gravitonの固定4GiBメモリでCPUクレジット枯渇の心配が少なく、負荷テスト結果が安定する |
| Performance Insights | 無効 | 有効（無料枠7日間保持） | staging同様に有効の想定 | 負荷テスト結果の分析用途 |
| CloudFront | 有効（キャッシュによるレイテンシ削減） | 同左 | 同左 | すでに実装済み |

---

## 3. 運用・保守性

| 項目 | 内容 |
|------|------|
| IaC | Terraform によりすべてのリソースを管理。再現性・バージョン管理を担保 |
| インフラCI/CD | GitHub Actions（mainブランチへのpushでterraform plan/apply）。account_group（nonprod/prod）とenvのペアでmatrix管理し、GitHub Environmentsでaccount_groupごとにAWS認証ロールを切り替え |
| アプリCI/CD | GitHub Actions（mainブランチへのpushでECRプッシュ・ECSデプロイ）。dev/staging/productionへのデプロイ対象・昇格ルールは下表参照 |
| ログ管理 | ECSタスクのログをCloudWatch Logsに収集。**保持期間はdev=7日／staging=30日／production=90日（予定）の3段階** |
| 監視 | CloudWatch アラームでECS・ALB・RDSのメトリクスを監視。SNS経由でメール通知。stagingではECSの`RunningTaskCount`にAuto Scalingの挙動を可視化するアラーム/ダッシュボードを追加 |
| コスト管理 | cost-stop（停止）・cost-start（起動）ワークフローで無駄なコストを削減。AWS Budgets（月$10アラート）・Cost Anomaly Detectionで異常課金を早期検知。負荷テスト用のRDSスペック一時変更もAWS CLIで同様に管理 |
| 命名規則 | `tomario-{env}-{リソース種別}` で統一。envは`dev`/`staging`/`shared`/`production`の4種類（`shared`はnonprodアカウント共通のsecurity/cost/ECRリソース用） |
| タグ管理 | `Project` / `Environment` / `ManagedBy` を全リソースに付与 |
| アカウント構成 | nonprod（dev・staging・shared）とprod（production）の2アカウント構成へ移行中。現状はnonprod側のディレクトリ・state分離（`envs/nonprod/*`）まで完了、prodアカウントは未作成 |
| デプロイ対象ルール | dev：任意のコミットを自動デプロイ。staging：リリース候補相当のタグでのみデプロイ、digestを記録。production：stagingで記録したdigestのみをpromoteジョブでpull→re-tag→push（再ビルドしない） |

---

## 4. セキュリティ

| 項目 | dev | staging | production（将来・方向性のみ） | 判断理由 |
|------|-----|---------|---|---|
| ネットワーク分離 | ECS・RDSをプライベートサブネットに配置 | 同左 | 同左 | インターネットからの直接アクセス不可 |
| 認証情報管理 | Secrets ManagerでDB認証情報を管理 | 同左 | 同左 | コードへのハードコードを排除 |
| AWS認証 | GitHub Actions OIDC（アクセスキー不使用） | 同左 | 同左（別アカウントの別ロール） | 長期認証情報のリスク排除 |
| データ暗号化 | RDS暗号化有効・S3 AES256暗号化 | 同左 | 同左 | 保存時暗号化を実施 |
| ECRイメージ | IMMUTABLEタグ・プッシュ時スキャン有効。dev/stagingは共通の`tomario-app`リポジトリを共有（nonprod/shared配下） | 同左 | production専用の`tomario-production-app`（クロスアカウント共有なし） | イメージの改ざん防止・脆弱性検出 |
| bootstrap_image | `tomario-app`（プライベートECR）の`bootstrap`タグを参照 | 同左 | 未定 | 従来のpublic ECR Gallery参照はNAT Gateway無しのVPCから到達不可でクラッシュループを起こしたため修正（2026-07-10） |
| IAM | 最小権限ポリシー（インフラ用・アプリデプロイ用で分離） | 同左 | 同左 | 職務分掌の実現 |
| HTTPS | CloudFrontでHTTPS終端（デフォルト証明書） | 同左 | 同左（独自ドメイン・ACM証明書は検討中・後回し） | 転送時暗号化 |
| 脅威検知 | GuardDuty有効化。**nonprod/shared配下でアカウント単位に集約管理**（dev単体ではなくアカウント共通） | shared側を共有 | 有効化予定（production独自） | GuardDutyはアカウント/リージョンにつき1つまでのため、workload環境ごとの重複を避けてsharedに集約 |
| Config / SecurityHub | 無効（nonprod/sharedでも無効。コスト・優先度の理由） | 同左 | 有効化予定 | 非商用のnonprodでCIS準拠チェックまで行う必要性が薄いため、production限定とする |
| ネットワーク監視 | VPC Flow Logs有効化（CloudWatch Logs・保持期間はログ保持日数の方針に準ずる） | 同左 | 同左 | 不正アクセスの痕跡調査に使用 |
| 監査ログ | CloudTrail有効化（管理イベント・S3保存）。**nonprod/shared配下でアカウント単位に集約管理** | shared側を共有 | production独自で有効化予定 | 追加コストほぼゼロ、GuardDutyと同じ理由でsharedに集約 |
| WAF | なし（cost-stop対象外のためstagingでも今回は導入しない） | なし | **導入する（2026-07-11方針転換）** | 当初は非商用のため見送っていたが、Web ACLは「存在（課金）／削除（無課金）」の時間按分課金と判明し、cost-stop/startと同じ発想で使う時だけ作る運用にすれば実質数十〜百円/月に収まるため導入方針に変更。実装は別途todo管理 |

---

## 5. 移行性

本システムは新規構築のため移行性は対象外とする。既存システムからのデータ移行・並行稼動・切り戻し手順の定義は不要。

---

## 6. 環境・エコロジー

### 6.1 システム環境

| 項目 | 内容 |
|------|------|
| AWSアカウント構成 | nonprod（dev・staging・shared）／prod（production）の2アカウント構成へ移行中。現状はnonprodアカウント内でのディレクトリ・state分離まで完了、prodアカウントは未作成 |
| リージョン | ap-northeast-1（東京）。対象ユーザーが日本国内のため低レイテンシを優先 |
| ネットワーク構成 | VPC + パブリック/プライベートサブネット（2AZ）。NAT Gatewayは導入しない（コスト対効果が悪いため。ECR/Secrets Manager/CloudWatch Logs/S3はVPCエンドポイント経由で完結する設計） |
| ドメイン管理 | 独自ドメイン・ACM証明書・Route53は検討中・後回し（現時点はCloudFrontデフォルトドメインを使用。技術的な必要性が薄く、他設計への影響もないためいつでも後付け可能） |

### 6.2 ライセンス管理

| ソフトウェア | ライセンス | 備考 |
|------------|-----------|------|
| Python / Flask | MIT / BSD | 商用利用可・無償 |
| MySQL（RDS） | GPL / 商用 | AWSマネージドのため個別ライセンス不要 |
| Terraform | BSL 1.1 | 商用製品への組み込み以外は無償 |
| GitHub Actions | GitHub利用規約 | パブリックリポジトリは無償 |

### 6.3 サステナビリティ

| 項目 | 内容 |
|------|------|
| 夜間・休日停止 | cost-stopワークフローでECS・RDSを停止。不要な電力消費を削減 |
| 最小リソース構成 | ECS CPU:256/Memory:512・RDS db.t3.micro。過剰なリソース確保をしない |
| コンテナ活用 | ECS Fargateによりサーバーレス運用。EC2と比べアイドル時のリソース無駄が少ない |
| データライフサイクル | ECRライフサイクルポリシーで古いイメージを自動削除。不要なストレージを削減 |
| CloudFrontキャッシュ | オリジン（ECS・S3）へのリクエスト数をキャッシュで削減 |

---

## 7. コスト最適化

| 項目 | 内容 |
|------|------|
| 夜間・休日停止 | cost-stopワークフローでECS・RDSを停止（dev・staging共通） |
| 最小インスタンス | ECS CPU:256/Memory:512・RDS db.t3.micro で最小構成（dev・staging共通、垂直スケールはしない方針） |
| 負荷テスト時の一時スケールアップ | RDSインスタンスクラスのみ、負荷テスト直前にAWS CLIで`db.t4g.medium`へ一時変更し、終了後に`db.t3.micro`へ戻す運用（Terraformの値は変更しない） |
| RDS Multi-AZ | 検討中・見送り（インスタンス料金がほぼ倍になるため、非商用ポートフォリオでは費用対効果が薄いと判断。将来変更の可能性あり） |
| ECRライフサイクル | 最新5世代のみ保持（古いイメージを自動削除） |
| VPCエンドポイント | プライベートサブネットからNAT Gateway不要でAWSサービスにアクセス（NAT Gatewayは月$30〜45程度の固定費がかかるため見送り） |
| ストレージ | RDS gp3（gp2より低コスト） |
| WAFのコスト最適化 | Web ACLは時間按分課金かつ「停止」ができないため、cost-stop/startと同様に使う時だけ作成・削除する運用を予定（常時起動なら月$14〜16程度だが、実運用ではごく小さく抑えられる見込み） |
| コスト監視 | AWS Budgets（月$10超過でメール通知）・Cost Anomaly Detection（$5以上の異常増加を翌日通知） |
| 想定コスト（dev・稼働時） | 約$3〜5/日（ECS・RDS・CloudFront・その他） |
