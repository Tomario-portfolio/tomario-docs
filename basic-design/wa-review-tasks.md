# WA レビュー 改善タスク一覧

ソース: `wa-review-report.md`（2026-06-30 レビュー）

## 凡例

| 分類 | 意味 |
|------|------|
| 🆕 サービス追加 | 新しいAWSサービス・リソースの追加が必要 |
| 🔧 修正 | 既存のコード・設計書・設定の変更で対応できる |
| 🏭 prod環境のみ | dev環境では不要。prod設計時に対応する |
| 🌐 dev・prod共通 | dev環境から導入する |

## ステータス

| 記号 | 意味 |
|------|------|
| ⬜ 未着手 | |
| 🔄 進行中 | |
| ✅ 完了 | |
| 🔒 保留 | コスト・優先度の理由で保留中 |

---

## 運用上の優秀性

| # | 重要度 | タスク | 分類 | 対象環境 | ステータス |
|---|--------|--------|------|---------|---------|
| OPS-1 | 中 | ~~デプロイロールバック手順を設計書（compute-high-level-spec.md）に追記する~~ | 🔧 修正 | dev・prod共通 | ✅ |
| OPS-2 | 低 | CloudWatch EMFによるアプリカスタムメトリクス収集を検討する（予約件数・レスポンスタイム） | 🆕 サービス追加 | 🏭 prod環境のみ | ⬜ |
| OPS-3 | 低 | CodeDeploy + ECS Blue/Greenデプロイを導入する | 🆕 サービス追加 | 🏭 prod環境のみ | ⬜ |
| OPS-4 | 低 | stg環境の構成をprd設計と並行して計画する | 🆕 サービス追加 | 🏭 prod環境のみ | ⬜ |

---

## セキュリティ

| # | 重要度 | タスク | 分類 | 対象環境 | ステータス |
|---|--------|--------|------|---------|---------|
| SEC-1 | 中 | ~~CloudTrailを有効化しS3に証跡を保存するTerraformリソースを追加する（管理イベントのみ・追加コストほぼゼロ）~~ | 🆕 サービス追加 | 🌐 dev・prod共通 | ✅ |
| SEC-2 | 中 | ~~GuardDutyを有効化する（30日無料試用。試用後は月~$1〜3）~~ | 🆕 サービス追加 | 🌐 dev・prod共通 | ✅ |
| SEC-3 | 中 | ~~VPC Flow LogsをCloudWatch Logsへ送信する（保持期間7日・低トラフィックなら月数十円）~~ | 🆕 サービス追加 | 🌐 dev・prod共通 | ✅ |
| SEC-4 | 低 | GitHub ActionsにPythonライブラリ脆弱性チェック（pip-audit）を追加する | 🔧 修正 | 🌐 dev・prod共通 | ⬜ |
| SEC-5 | 低 | AWS WAFをCloudFront・ALBに適用する（WebACL固定$5/月） | 🆕 サービス追加 | 🏭 prod環境のみ | ⬜ |
| SEC-6 | 低 | Security Hub + CIS AWS Foundations Benchmarkを有効化してCIS準拠スコアを可視化する（AWS Config必要・月数百円〜） | 🆕 サービス追加 | 🏭 prod環境のみ | 🔒 |

### SEC-6 補足（Security Hub + CIS）

会社でCIS準拠を意識しているため将来的に取り入れたい。  
Security Hubを有効化してCIS標準を選択することでCISコントロールの自動チェックが可能になる。  
SEC-1（CloudTrail）とAWS Configの有効化が前提条件。

```
Security Hub
    ├── CloudTrail（必須）← SEC-1で対応
    ├── AWS Config（必須・コスト発生）
    └── GuardDuty（オプション）← SEC-2で対応
```

---

## 信頼性

| # | 重要度 | タスク | 分類 | 対象環境 | ステータス |
|---|--------|--------|------|---------|---------|
| REL-1 | 中 | ~~ECSタスク数1はdev環境の許容範囲として設計書（compute-high-level-spec.md）に明記する~~ | 🔧 修正 | dev・prod共通 | ✅ |
| REL-2 | 中 | ECS Application Auto Scalingを定義する（CPU 70%以上でスケールアウト） | 🆕 サービス追加 | 🏭 prod環境のみ | ⬜ |
| REL-3 | 低 | ~~RDSスナップショットからのリストアテスト手順を設計書（backup-high-level-spec.md）に追記する~~ | 🔧 修正 | dev・prod共通 | ✅ |
| REL-4 | 低 | Lambda + EventBridgeでRDS定期自動停止を実装する（7日自動起動問題の解消） | 🆕 サービス追加 | 🌐 dev・prod共通 | ⬜ |

---

## パフォーマンス効率

| # | 重要度 | タスク | 分類 | 対象環境 | ステータス |
|---|--------|--------|------|---------|---------|
| PERF-1 | 低 | CloudWatchでECS CPU使用率を数週間観測し、0.25vCPU/0.5GBの妥当性を検証する | 🔧 修正 | dev・prod共通 | ⬜ |
| PERF-2 | 低 | ECS FargateタスクをGraviton（ARM64）に変更する（コスト約20%削減・電力効率向上） | 🔧 修正 | 🌐 dev・prod共通 | ⬜ |
| PERF-3 | 低 | Locustまたはab（Apache Bench）でdev環境の簡易負荷テストを実施し結果を設計書に記録する | 🔧 修正 | dev・prod共通 | ⬜ |
| PERF-4 | 低 | RDS Proxyの導入を検討する（ECSタスク複数台時のDB接続数管理） | 🆕 サービス追加 | 🏭 prod環境のみ | ⬜ |

---

## コスト最適化

| # | 重要度 | タスク | 分類 | 対象環境 | ステータス |
|---|--------|--------|------|---------|---------|
| COST-1 | 中 | ~~AWS Budgets（月$10アラート）をTerraformで定義する（無料）~~ | 🆕 サービス追加 | 🌐 dev・prod共通 | ✅ |
| COST-2 | 中 | ~~Cost Anomaly Detectionモニターを作成しSNS通知を設定する（無料）~~ | 🆕 サービス追加 | 🌐 dev・prod共通 | ✅ |
| COST-3 | 低 | ~~TerraformのproviderブロックにProject/Environment/ManagedByのdefault_tagsを追加する~~ | 🔧 修正 | 🌐 dev・prod共通 | ✅ |
| COST-4 | 低 | Lambda + EventBridgeでRDS定期自動停止を実装する（REL-4と同一タスク） | 🆕 サービス追加 | 🌐 dev・prod共通 | ⬜ |

---

## サステナビリティ

| # | 重要度 | タスク | 分類 | 対象環境 | ステータス |
|---|--------|--------|------|---------|---------|
| SUS-1 | 低 | ECS FargateタスクをGraviton（ARM64）に変更する（PERF-2と同一タスク） | 🔧 修正 | 🌐 dev・prod共通 | ⬜ |
| SUS-2 | 低 | S3フロントエンドバケットにライフサイクルポリシー（90日後にIntelligent-Tiering移行）を追加する | 🔧 修正 | 🌐 dev・prod共通 | ⬜ |
| SUS-3 | 低 | RDS 7日自動起動問題を解消する（REL-4・COST-4と同一タスク） | 🆕 サービス追加 | 🌐 dev・prod共通 | ⬜ |

---

## 重複タスクの統合

以下のタスクは同一の実装で複数の柱の課題を解消できる。

| 統合タスク | 解消される項目 |
|-----------|--------------|
| Lambda + EventBridgeでRDS定期自動停止を実装する | REL-4 / COST-4 / SUS-3 |
| ECS FargateタスクをGraviton（ARM64）に変更する | PERF-2 / SUS-1 |

---

## 着手優先度まとめ

### 今すぐ着手できる（コストゼロ・コード修正のみ）

1. ~~`COST-3` Terraformのdefault_tags追加~~ ✅ 実装済み確認
2. ~~`OPS-1` ロールバック手順を設計書に追記~~ ✅
3. ~~`REL-1` ECSタスク数1をdev許容範囲として設計書に明記~~ ✅
4. ~~`REL-3` リストアテスト手順を設計書に追記~~ ✅

### 次に着手（サービス追加・コスト低）

5. ~~`SEC-1` CloudTrail有効化（無料）~~ ✅
6. ~~`COST-1` AWS Budgets定義（無料）~~ ✅
7. ~~`COST-2` Cost Anomaly Detection設定（無料）~~ ✅
8. ~~`SEC-3` VPC Flow Logs追加（月数十円）~~ ✅
9. `SEC-4` pip-auditをCI/CDに追加
10. `REL-4 / COST-4 / SUS-3` Lambda + EventBridgeでRDS自動停止

### prod設計時に対応

11. `SEC-2` GuardDuty有効化
12. `SEC-5` AWS WAF導入
13. `SEC-6` Security Hub + CIS準拠チェック（CloudTrail + AWS Config必須）
14. `REL-2` ECS Auto Scaling設定
15. `OPS-2` CloudWatch EMFカスタムメトリクス
16. `OPS-3` Blue/Greenデプロイ
17. `PERF-4` RDS Proxy導入
