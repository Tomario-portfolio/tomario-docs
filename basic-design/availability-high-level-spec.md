# 可用性

## システム重要度

本システムはホテル予約サービスを提供するため、以下の重要度とする。

| 環境 | 重要度 | 目標月間稼働率 |
|------|--------|-------------|
| production | Medium | 99.9%（常時稼働に切り替えた後の目標。マルチリージョン構成は採用せず、2AZ内の冗長化で現実的に狙える水準として設定） |
| staging | Low〜Medium | 目標なし（負荷テスト・障害試験の検証環境） |
| dev | Low | 目標なし（手動リカバリー） |

## 稼働時間

production環境は本来24時間稼働を想定するが、**一般公開前はコスト優先でdev・staging同様のcost-stop運用とする**。公開時に常時稼働へ切り替える。
dev・staging環境はコスト最適化のため作業時のみ起動する運用とする（cost-stop/start）。

## 冗長構成

### ALB + ECS Serviceによる冗長

ALBを上段に配置してECSタスクへトラフィックを転送する。
ALBのヘルスチェックにより異常なタスクへのルーティングを自動的に停止する。
ECSサービスがタスクの死活を監視し、異常終了時に自動で再起動する。

| 環境 | タスク数 | Auto Scaling |
|------|--------|-------------|
| dev | 1 | 無効 |
| staging | 2（2AZに分散） | 有効（min=2/max=4、target CPU 70%） |
| production | 2（2AZに分散） | 有効（min=2/max=4、target CPU 70%） |

staging・productionはタスクを2AZに分散配置するため、1AZ障害時も残りのタスクでサービスを継続できる。
CPU/メモリは全環境256/512とし、垂直スケールはせず水平（タスク数）のみで負荷に対応する（詳細は[compute-high-level-spec.md](compute-high-level-spec.md)参照）。

### RDS

全環境Single-AZ構成とする。
Multi-AZは変数化済みだが、全環境で無効としている。判断の経緯は[ADR: RDS Multi-AZの見送り](../../tomario-steering/adr/infra/database/003-multi-az-cost-tradeoff.md)を参照。

そのためDB層はAZ障害時の単一障害点として残る。AZ障害時はポイントインタイムリストアで別AZに復元する運用で対応する（[backup-high-level-spec.md](backup-high-level-spec.md)参照）。productionで可用性要件が高まった場合は、`multi_az`変数を`true`にするだけで自動フェイルオーバー構成に切り替えられる。

## リージョン間の冗長

リージョンを跨いだ冗長構成は採用しない。
DR要件が発生した場合に改めて検討する。
