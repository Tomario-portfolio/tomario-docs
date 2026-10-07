# database 環境定義

RDS・DBサブネットグループ・RDS用セキュリティグループを管理するコンポーネント（`modules/database`）と、RDS自動停止Lambda（`modules/rds-autostop`）を定義する。

RDSはcost-stopの対象（未使用時は停止）。productionも一般公開前は同じ扱いとする。
設計判断（Multi-AZの見送り・インスタンスクラス・Performance Insightsの見送り）の理由は[database-high-level-spec.md](../basic-design/database-high-level-spec.md)の「関連する設計判断」に挙げたADRを参照。

---

## セキュリティグループ

### RDS-SG

| 項目 | dev | staging | production |
|------|-----|---------|---|
| SG名 | tomario-dev-rds-sg | tomario-staging-rds-sg | tomario-production-rds-sg |

| 方向 | プロトコル | ポート | 送信元 |
|------|----------|-------|--------|
| インバウンド | TCP | 3306 | ECS-SG |
| アウトバウンド | 全て | 全て | 0.0.0.0/0 |

---

## DBサブネットグループ

| 項目 | dev | staging | production |
|------|-----|---------|---|
| サブネットグループ名 | tomario-dev-rds-subnet-group | tomario-staging-rds-subnet-group | tomario-production-rds-subnet-group |
| 対象サブネット | プライベートサブネット×2（1a・1c） | プライベートサブネット×2（1a・1c） | プライベートサブネット×2（1a・1c） |

---

## RDS

| 項目 | dev | staging | production |
|------|-----|---------|---|
| インスタンス名 | tomario-dev-rds | tomario-staging-rds | tomario-production-rds |
| インスタンスクラス | db.t3.micro | db.t3.micro | db.t3.micro |
| エンジン | MySQL 8.4 | MySQL 8.4 | MySQL 8.4 |
| DB名 | tomario | tomario | tomario |
| マスターユーザー名 | admin | admin | admin |
| ストレージタイプ | gp3 | gp3 | gp3 |
| ストレージサイズ（allocated） | 20GB | 20GB | 20GB |
| ストレージ自動拡張（max_allocated_storage） | 未設定 | 未設定 | 未設定 |
| ストレージ暗号化 | 有効 | 有効 | 有効 |
| ポート | 3306 | 3306 | 3306 |
| Multi-AZ | 無効（`multi_az=false`） | 無効 | 無効 |
| 自動マイナーバージョンアップ | 有効 | 有効 | 有効 |
| Performance Insights | 無効 | 無効 | 無効 |
| 削除保護（`deletion_protection`） | 無効（固定値） | 無効 | 無効 |
| skip_final_snapshot | true（固定値） | true | true |

stagingでの負荷テスト時のみ、インスタンスクラスを一時的に`db.t4g.medium`へ変更する（Terraformの値は変えない）。手順は`tomario-steering/verification/non-functional-test/procedures/performance-test-procedure.md`を参照。

### バックアップ

| 項目 | dev | staging | production |
|------|-----|---------|---|
| 自動バックアップ | 有効 | 有効 | 有効 |
| バックアップ保持期間 | 7日 | 7日 | 7日 |
| バックアップウィンドウ | 18:00〜19:00 UTC | 18:00〜19:00 UTC | 18:00〜19:00 UTC |
| メンテナンスウィンドウ | 日曜 19:00〜20:00 UTC | 日曜 19:00〜20:00 UTC | 日曜 19:00〜20:00 UTC |

RPO/RTO目標とリストア方式は[backup-high-level-spec.md](../basic-design/backup-high-level-spec.md)、リストア手順と訓練結果は`tomario-steering/verification/non-functional-test/`配下のbackup-test手順書・結果を参照。

---

## Secrets Manager（DB認証情報）

| 項目 | 内容 |
|------|------|
| 管理方法 | `manage_master_user_password`によるSecrets Manager自動管理 |
| ローテーション | 自動スケジュールは未設定（CLIの`--rotate-master-user-password`で手動実行可能） |
| 取得権限 | ECS Task Execution Roleに`secretsmanager:GetSecretValue`を付与（[backend-environment-design.md](backend-environment-design.md)参照） |

---

## RDS自動停止Lambda（`modules/rds-autostop`）

停止したRDSが7日後にAWSにより自動起動された場合に、再停止する。

| 項目 | dev | staging | production |
|------|-----|---------|---|
| 配置 | あり | あり | あり |
| 起動スケジュール | EventBridge `rate(1 hour)`（1時間ごと） | 同左 | 同左 |
| 停止条件 | RDSが`available`、かつ直近180分以内に7日制約による自動起動のRDSイベントがあり、かつ対応するECSサービスが稼働していない | 同左 | 同左 |

---

## 未解決事項

| 項目 | 内容 |
|------|------|
| skip_final_snapshot | 全環境`true`固定のため、意図せずdestroyした場合に最終スナップショットが残らない。cost-stopは停止のみでdestroyしないため通常運用では影響しないが、productionで`false`にするか要検討 |
| max_allocated_storage | 全環境未設定（20GB固定）。ストレージ枯渇時に自動拡張されないため、productionで設定するか要検討 |
