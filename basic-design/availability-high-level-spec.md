# 可用性

## 基本方針

アプリケーション層（ALB + ECS）は2AZに冗長化し、タスク障害・AZ障害時もサービスを継続できる構成とする。
DB層（RDS）はSingle-AZとし、障害時はバックアップからの復旧で対応する。
リージョンを跨いだ冗長構成は採用しない。

## システム重要度

| 環境 | 重要度 | 目標月間稼働率 |
|------|--------|-------------|
| production | Medium | 99.9%（2AZ内の冗長化で狙える水準） |
| staging | Low〜Medium | 目標なし（負荷テスト・障害試験の検証環境） |
| dev | Low | 目標なし（手動リカバリー） |

productionは常時稼働とする。dev・stagingは使う時だけ起動する運用とする（[cost-high-level-spec.md](cost-high-level-spec.md)参照）。

## 冗長構成

### アプリケーション層（ALB + ECS Service）

```
        ALB（2AZのパブリックサブネット）
         │ヘルスチェックで異常タスクを切り離し
   ┌─────┴─────┐
ECSタスク(1a)   ECSタスク(1c)   ← ECSサービスが異常終了したタスクを自動再起動
```

- ALBのヘルスチェックにより、異常なタスクへのルーティングを自動的に停止する
- ECSサービスがタスクの死活を監視し、異常終了時に自動で再起動する
- staging・productionはタスクを2AZに分散配置するため、1AZ障害時も残りのタスクでサービスを継続できる
- devは単一タスクとし、タスク再起動までの短時間のダウンタイムを許容する
- 負荷増加時はAuto Scalingでタスク数を水平に増やす

タスク数・スケーリング設定の具体値は[環境定義書](../environment-definitions/backend-environment-design.md)を参照。

### DB層（RDS）

全環境Single-AZ構成とし、DB層はAZ障害時の単一障害点として残ることを許容する。
AZ障害時はポイントインタイムリストアで別AZに復元して対応する（[backup-high-level-spec.md](backup-high-level-spec.md)参照）。
Multi-AZは変数化しており、可用性要件が高まった場合は設定変更で自動フェイルオーバー構成に切り替えられる。

Single-AZとした理由は[ADR: RDS Multi-AZの見送り](../../tomario-steering/adr/infra/database/003-multi-az-cost-tradeoff.md)を参照。

## リージョン間の冗長

リージョンを跨いだ冗長構成は採用しない。
DR要件が発生した場合に改めて検討する。
