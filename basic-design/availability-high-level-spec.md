# 可用性

## 要件

## システム重要度

本システムはホテル予約サービスを提供するため、以下の重要度とする。

| 環境 | 重要度 | 目標月間稼働率 |
|------|--------|-------------|
| production | Medium | 99.9%（常時稼働に切り替えた後の目標。マルチリージョン構成は採用せず、2AZ内の冗長化で現実的に狙える水準として設定） |
| staging | Low〜Medium | 目標なし（負荷テスト・障害試験の検証環境） |
| dev | Low | 目標なし（手動リカバリー） |

## 稼働時間

production環境は本来24時間稼働を想定するが、**リリース前（面接活動期間に入るまで）はコスト優先でdev・staging同様のcost-stop運用とする**。リリース時（面接日程が近づいたタイミング）に常時稼働へ切り替える。切り替えが必要な設定（ALB削除保護など）はTerraformコード上にコメントで両方の値を用意しておき、リリース時はコメントを入れ替えるだけで切り替えられるようにする（詳細は[backend-environment-design.md](../../tomario-infra-design/environment-definitions/backend-environment-design.md)参照）。
dev・staging環境はコスト最適化のため作業時のみ起動する運用とする（cost-stop/start）。

## 冗長構成

### ALB + ECS Serviceによる冗長

ALBを上段に配置してECSタスクへトラフィックを転送する。
ALBのヘルスチェックにより異常なタスクへのルーティングを自動的に停止する。
ECSサービスがタスクの死活を監視し、異常終了時に自動で再起動する。

dev環境ではタスク数を1（`desired_count=1`）とする。
staging環境ではタスク数を2（`desired_count=2`）とし、Application Auto Scaling（min=2/max=4、target CPU 70%）を有効化する。水平スケーリングの実挙動を検証することが目的で、CPU/メモリ自体はdevと同じ値（256/512）のまま変えない（垂直スケールはしない方針）。
production環境はタスク数・Auto Scaling設定（min=2/max=4、target CPU 70%）はstagingでの検証結果を引き継ぐ想定だが、CPU/メモリはstagingの2倍（512/1024）とする。

<!--
### ALB + ASGによる冗長（旧）

ALBを上段に配置してEC2をActive-Active構成とする。
ASGにより2AZにまたがってEC2インスタンスを管理し、1AZ障害時も残りのAZでサービスを継続する。
ALBのヘルスチェックにより異常なインスタンスへのルーティングを自動的に停止する。
-->

### RDS

dev・staging環境ではコスト最適化のためSingle-AZ構成とする。
Multi-AZは変数化済みだが、dev・stagingでは値をオフのままとする（非商用ポートフォリオでインスタンス料金がほぼ倍になるコストに見合う必要性が薄いと判断）。
production環境はMulti-AZを有効化する。ALB/ECSの水平冗長化に加えてDB層も自動フェイルオーバーを持たせることで、単一障害点を残さない構成とする（詳細は[database-high-level-spec.md](database-high-level-spec.md)参照）。

## リージョン間の冗長

リージョンを跨いだ冗長構成は採用しない。
DR要件が発生した場合に改めて検討する。
