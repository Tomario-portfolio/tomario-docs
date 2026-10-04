# データベース

## 基本方針

データベースにはAWSマネージドサービスのRDSを使用する。
バックアップ・パッチ適用をAWSに委任することで運用コストを削減する。
RDSはプライベートサブネットに配置し、インターネットからの直接アクセスを遮断する。

## RDS

MySQL 8.4をエンジンとして使用する（当初は8.0だったが、標準サポート終了に伴い2026-07-07にアップグレード）。
ECSタスク上のFlaskアプリケーションからのみ接続を許可する。
ストレージはgp3・暗号化有効とする。

DB接続情報（ユーザー名・パスワード）は`manage_master_user_password`によりSecrets Managerで管理し、コードに直接記載しない。

### 環境別構成方針

| 項目 | dev | staging | production |
|------|-----|---------|-----------|
| インスタンスクラス | db.t3.micro | db.t3.micro | db.t3.micro |
| Multi-AZ | 無効 | 無効 | 無効（変数化済み、検討中） |
| Performance Insights | 無効 | 無効 | 無効 |
| 運用 | 作業時以外は停止 | 同左 | 一般公開前は同左 |

- インスタンスクラスは全環境`db.t3.micro`とする（垂直スケールはしない方針）。stagingで負荷テストを実施する直前のみ、AWS CLIで一時的に`db.t4g.medium`（Graviton、固定4GiBメモリ）にスケールアップし、終了後に`db.t3.micro`へ戻す運用とする。Terraform上の値は変更しない。選定理由は[ADR: RDSインスタンスクラスの選定](../../tomario-steering/adr/infra/database/005-db-instance-class-selection.md)を参照
- Multi-AZは変数（`multi_az`）化のみ行い、全環境で無効としている。判断の経緯は[ADR: RDS Multi-AZの見送り](../../tomario-steering/adr/infra/database/003-multi-az-cost-tradeoff.md)を参照
- Performance Insightsは導入していない。インスタンスクラスの制約で技術的に導入できなかった経緯は[ADR: RDS Performance Insightsの導入見送り](../../tomario-steering/adr/infra/database/004-performance-insights-instance-class-limitation.md)を参照

### RDS停止の7日制約への対策

RDSは停止しても7日経過するとAWSにより自動的に起動される。
これに対応するため、全環境にRDS自動停止Lambda（`modules/rds-autostop`）を配置している。
EventBridgeで毎日1回（JST 05:00）起動し、RDSが`available`なのに対応するECSサービスが稼働していない場合（＝cost-startではなく7日制約による自動起動と判断できる場合）にRDSを再停止する。

## スキーマ

| テーブル名 | 概要 |
|-----------|------|
| users | ユーザー情報（認証・予約との紐付け） |
| rooms | 部屋マスタ（部屋番号・タイプ・定員・料金） |
| bookings | 予約情報（ユーザー・部屋・チェックイン/アウト日・ステータス） |

詳細なカラム定義は[要件定義書](../requirements/requirements.md)に記載する。
