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
| インスタンス名 | tomario-dev-rds | tomario-staging-rds | tomario-production-rds |
| インスタンスクラス（通常時） | db.t3.micro | db.t3.micro（変更なし） | db.t3.micro（stagingで検証済みの値を踏襲。垂直スケールはしない方針を継続） |
| インスタンスクラス（負荷テスト時） | ― | 一時的に`db.t4g.medium`へAWS CLIでスケールアップ（`apply_immediately`、終了後に戻す。Terraformの値は変えない） | ― |
| エンジン | MySQL 8.4 | MySQL 8.4 | MySQL 8.4 |
| ストレージタイプ | gp3 | gp3 | gp3 |
| ストレージサイズ（allocated） | 20GB | 20GB | 20GB |
| ストレージ自動拡張（max_allocated_storage） | 未設定 | 未設定 | 未設定（固定20GBのまま。導入は検討課題） |
| ポート | 3306 | 3306 | 3306 |
| Multi-AZ | なし（Single-AZ、`multi_az`変数のデフォルトfalse） | なし（同左。コスト対効果が薄いため見送り） | なし（同左。`multi_az=true`にすればいつでも有効化可能だが、現状は未設定） |
| 自動マイナーバージョンアップ | 有効 | 有効 | 有効 |
| Performance Insights | 無効 | 無効 | 無効（`db.t3.micro`／`t3.small`／`t4g.micro`ではMySQL 8.4.9でPerformance Insights自体が未サポート。2026-09-29に共通モジュールへの追加を試みたが`terraform apply`が失敗し判明、`t4g.medium`以上への恒久的な引き上げはコスト方針に反するため見送り） |
| 削除保護（`deletion_protection`） | 無効（固定値、変数化していない） | 無効（同左） | 無効（同左） |
| 運用 | 未使用時は停止（~$0.23/月） | 未使用時は停止（負荷テスト実施時のみ起動） | 一般公開前：stagingと同じcost-stop対象（未使用時は停止、~$3/月はストレージ分）。公開後：常時稼働（db.t3.micro Single-AZで~$22/月） |

> **補足：** エンジンバージョンは当初MySQL 8.0だったが、標準サポート終了（2026-07-31）に伴い8.4へアップグレード済み（2026-07-07）。

### バックアップ

| 項目 | dev | staging | production |
|------|-----|---------|---|
| 自動バックアップ | 有効 | 有効 | 有効 |
| バックアップ保持期間 | 7日 | 7日 | 7日 |
| バックアップウィンドウ | 18:00〜19:00 UTC | 18:00〜19:00 UTC | 18:00〜19:00 UTC |
| メンテナンスウィンドウ | 日曜 19:00〜20:00 UTC | 日曜 19:00〜20:00 UTC | 日曜 19:00〜20:00 UTC |
| RPO/RTO目標 | 規定なし | 15分/30分（検証目的） | 15分/30分（`backup-high-level-spec.md`参照） |
| リストア訓練 | 未実施 | 実施済み（2026-07-21、実測RTO約14分） | 未実施（破壊的操作のためstaging限定で実施する方針。RTO/RPO目標値はstagingの実測を踏襲） |

---

## Secrets Manager

| 項目 | 内容 |
|------|------|
| 管理方法 | `manage_master_user_password` によるSecrets Manager自動管理 |
| ローテーション | 自動スケジュールでの定期ローテーションは未設定。CLIで手動トリガー可能（`--rotate-master-user-password`） |
| 取得権限 | ECS Task Execution Roleに `secretsmanager:GetSecretValue` を付与（backendモジュールで管理） |

> **注意：** RDSを停止しても7日後にAWSが自動で再起動する。全環境にRDS自動停止Lambda（`modules/rds-autostop`）を配置し、EventBridgeで毎日1回、ECSが停止しているのにRDSだけ`available`な状態を検知して自動的に再停止する運用に切り替え済み（2026-09-29、PR #93。手動での毎週停止し直す運用は廃止）。

---

## TBD解消事項

| 項目 | 内容 |
|------|------|
| DB名 | `tomario`（Terraformデフォルト値として設定済み） |
| マスターユーザー名 | `admin`（Terraformデフォルト値として設定済み） |
| skip_final_snapshot | 全環境`true`（固定値、変数化していない）。production含め、destroy時に最終スナップショットは残らない。cost-stopはRDSを`stop-db-instance`するだけでdestroyしないため通常運用では影響しないが、意図せずdestroyした場合のデータ消失リスクとしては残っている（検討課題） |
