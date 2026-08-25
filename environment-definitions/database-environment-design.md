# database 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

RDS・DBサブネットグループ・Secrets Managerを管理するコンポーネント。
RDSはプライベートサブネットに配置し、ECSタスクからのみ接続を許可する。
コスト管理のため、未使用時はRDSを停止する運用とする。

---

## セキュリティグループ

### RDS-SG

| 項目 | dev | staging | production |
|------|-----|---------|---|
| SG名 | tomario-dev-rds-sg | tomario-staging-rds-sg | tomario-production-rds-sg |

| 方向 | プロトコル | ポート | 送信元 |
|------|----------|-------|--------|
| インバウンド | TCP | 3306 | ECS-SG（backendモジュールで管理） |
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
| インスタンス名 | tomario-dev-rds | tomario-staging-rds | tomario-production-rds（予定） |
| インスタンスクラス（通常時） | db.t3.micro | db.t3.micro（変更なし） | db.t3.micro（stagingで検証済みの値を踏襲。垂直スケールはしない方針を継続） |
| インスタンスクラス（負荷テスト時） | ― | 一時的に`db.t4g.medium`へAWS CLIでスケールアップ（`apply_immediately`、終了後に戻す。Terraformの値は変えない） | ― |
| エンジン | MySQL 8.4 | MySQL 8.4 | MySQL 8.4 |
| ストレージタイプ | gp3 | gp3 | gp3 |
| ストレージサイズ（allocated） | 20GB | 20GB | 20GB |
| ストレージ自動拡張（max_allocated_storage） | なし | なし | **100GB**（AWS推奨のストレージオートスケーリングを有効化し、想定外のディスク枯渇を防ぐ） |
| ポート | 3306 | 3306 | 3306 |
| Multi-AZ | なし（Single-AZ、変数化のみ） | なし（Single-AZ、変数化のみ。コスト対効果が薄いため見送り） | **あり**（Multi-AZ有効化。自動フェイルオーバーによる信頼性向上、コストはSingle-AZの約2倍） |
| 自動マイナーバージョンアップ | 有効 | 有効 | 有効 |
| Performance Insights | 無効 | 無効（Wave B対応、`todo.md` #9で計画的に後回し） | 無効（staging同様Wave B。productionでもstagingでの検証後に導入） |
| 削除保護（`deletion_protection`） | 無効（cost-stopで定期的に削除するため） | 無効（同左） | **有効**（誤destroy防止、AWS推奨） |
| 運用 | 未使用時は停止（~$0.23/月） | 未使用時は停止（負荷テスト実施時のみ起動） | **リリース前：**stagingと同じcost-stop対象（未使用時は停止、Multi-AZストレージ分~$0.46/月）。**リリース後：**常時稼働（db.t3.micro Multi-AZで~$30/月） |

> **補足：** エンジンバージョンは当初MySQL 8.0だったが、標準サポート終了（2026-07-31）に伴い8.4へアップグレード済み（2026-07-07）。

### バックアップ

| 項目 | dev | staging | production |
|------|-----|---------|---|
| 自動バックアップ | 有効 | 有効 | 有効 |
| バックアップ保持期間 | 7日 | 7日 | 7日 |
| バックアップウィンドウ | 18:00〜19:00 UTC | 18:00〜19:00 UTC | 18:00〜19:00 UTC |
| メンテナンスウィンドウ | 日曜 19:00〜20:00 UTC | 日曜 19:00〜20:00 UTC | 日曜 19:00〜20:00 UTC |
| RPO/RTO目標 | 規定なし | 15分/30分（検証目的） | 15分/30分（`backup-high-level-spec.md`参照） |
| リストア訓練 | 未実施 | 実施済み（2026-07-21、実測RTO約14分） | production構築後、同じ訓練を実施し目標値内に収まることを確認する |

---

## Secrets Manager

| 項目 | 内容 |
|------|------|
| 管理方法 | `manage_master_user_password` によるSecrets Manager自動管理 |
| ローテーション | AWSによる自動ローテーション |
| 取得権限 | ECS Task Execution Roleに `secretsmanager:GetSecretValue` を付与（backendモジュールで管理） |

> **注意：** RDSを停止しても7日後にAWSが自動で再起動する。毎週手動で停止し直す運用とする。

---

## TBD解消事項

| 項目 | 内容 |
|------|------|
| DB名 | `tomario`（Terraformデフォルト値として設定済み） |
| マスターユーザー名 | `admin`（Terraformデフォルト値として設定済み） |
| skip_final_snapshot | dev・staging：true（スナップショット不要）。production：**false**に決定（誤destroy時もデータを失わないよう、最終スナップショットを必ず残す） |
