# データベース

## 基本方針

データベースにはAWSマネージドサービスのRDS（MySQL）を使用する。
バックアップ・パッチ適用をAWSに委任することで運用負荷を削減する。
選定理由は[ADR: データベース基盤選択](../../tomario-steering/adr/infra/database/001-managed-database-selection.md)・[ADR: データベースエンジン選択](../../tomario-steering/adr/infra/database/002-mysql-engine-selection.md)を参照。

## 役割

予約システムの永続データ（ユーザー・客室・予約）を保持する。
同一客室・同一期間の二重予約を防ぐため、トランザクション整合性をDB側で担保する。

## 配置・接続方式

RDSはプライベートサブネット（2AZにまたがるDBサブネットグループ）に配置し、インターネットからの直接アクセスを遮断する。
接続を許可するのはECSタスク上のFlaskアプリケーションのみとする。

```
ECSタスク（ECS-SG）──MySQL(3306)──▶ RDS（RDS-SG：ECS-SGからのみ許可）
```

## セキュリティ境界

| 観点 | 方針 |
|------|------|
| ネットワーク | RDS-SGでECS-SGからのMySQLポートのみ許可する |
| 認証情報 | `manage_master_user_password`によりSecrets Managerで管理し、コードに直接記載しない。アプリはECSタスク起動時に環境変数として受け取る |
| 保存データ | ストレージ暗号化を有効にする |

## 可用性・バックアップ

全環境Single-AZ構成とし、AZ障害時はポイントインタイムリストアで復旧する（[availability-high-level-spec.md](availability-high-level-spec.md)・[backup-high-level-spec.md](backup-high-level-spec.md)参照）。
Multi-AZを見送った理由は[ADR: RDS Multi-AZの見送り](../../tomario-steering/adr/infra/database/003-multi-az-cost-tradeoff.md)を参照。

## 関連する設計判断

| 判断 | ADR |
|------|-----|
| インスタンスクラスの選定（負荷テスト時の一時スケールアップを含む） | [ADR: RDSインスタンスクラスの選定](../../tomario-steering/adr/infra/database/005-db-instance-class-selection.md) |
| Performance Insightsの導入見送り | [ADR: RDS Performance Insightsの導入見送り](../../tomario-steering/adr/infra/database/004-performance-insights-instance-class-limitation.md) |
| 未使用時の停止運用（7日自動起動への対策を含む） | [ADR: cost-stop/startによる使う時だけ起動する運用](../../tomario-steering/adr/infra/cost/001-cost-stop-start-operation.md) |

エンジンバージョン・インスタンスクラス・ストレージ・バックアップ設定等の具体値は[環境定義書](../environment-definitions/database-environment-design.md)を参照。

## スキーマ

| テーブル名 | 概要 |
|-----------|------|
| users | ユーザー情報（認証・予約との紐付け） |
| rooms | 客室マスタ（部屋番号・タイプ・定員・料金） |
| bookings | 予約情報（ユーザー・客室・チェックイン/アウト日・ステータス） |

詳細なカラム定義は[要件定義書](../requirements/requirements.md)に記載する。
