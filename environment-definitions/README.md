# 環境定義書 索引

## 本書群の位置付け

環境定義書は、各環境に**実際に作る構成（target state）**を具体値で定義する。リソース名・CIDR・セキュリティグループ・CPU/メモリ・インスタンスクラス・閾値・保持期間・コスト目安などを記載する。

以下は環境定義書では扱わず、それぞれの文書へ分離している。

| 内容 | 記載先 |
|------|-------|
| 構成・責務・方針 | [基本設計書](../basic-design/basic-design-high-level-spec.md) |
| なぜその構成・運用にしたか（設計判断の理由・経緯） | ADR（`tomario-steering/adr/infra/`） |
| 運用手順（cost-stop/start、リストア、負荷テスト時の一時スケールアップ等）と試験の実績 | `tomario-steering/verification/`・`tomario-steering/release-management/` |

---

## 共通事項

| 項目 | 内容 |
|------|------|
| Terraformバージョン | `>= 1.10`（`envs/`配下。`bootstrap-*`は`>= 1.5`） |
| AWSプロバイダー | hashicorp/aws `~> 6.0` |
| リージョン | ap-northeast-1（東京）。CloudFront用WAF・WAFアラームのみus-east-1 |

---

## 環境境界

| アカウント | 環境 | 種別 | 内容 |
|----------|------|------|------|
| nonprod | dev | workload | 開発環境 |
| nonprod | staging | workload | 負荷試験・障害試験の検証環境 |
| nonprod | shared | アカウント共通 | GuardDuty・CloudTrail・Budgets・Cost Anomaly Detection・ECRなど、アカウントに1つだけ置くリソース。VPCを持たない |
| prod | production | workload | 本番環境。shared相当の環境は持たず、アカウント共通リソースも`envs/prod/production`配下で管理する |

---

## モジュールと環境・定義書の対応

| モジュール（`tomario-infra/modules/`） | dev | staging | nonprod/shared | production | 環境定義書 |
|-------------------------------------|-----|---------|----------------|-----------|-----------|
| network | ○ | ○ | ― | ○ | [network](network-environment-design.md) |
| backend | ○ | ○ | ― | ○ | [backend](backend-environment-design.md) |
| ecr | ― | ― | ○ | ○ | [ecr](ecr-environment-design.md) |
| database | ○ | ○ | ― | ○ | [database](database-environment-design.md) |
| rds-autostop | ○ | ○ | ― | ○ | [database](database-environment-design.md) |
| frontend | ○ | ○ | ― | ○ | [frontend](frontend-environment-design.md) |
| logging | ○ | ○ | ○（CloudTrail用） | ○ | [logging](logging-environment-design.md) |
| monitoring | ○ | ○ | ― | ○ | [monitoring](monitoring-environment-design.md) |
| security | ― | ― | ○ | ○ | [security](security-environment-design.md) |
| waf | ― | ― | ― | ○（必要な期間のみ） | [security](security-environment-design.md) |
| cost | ― | ― | ○ | ○ | [cost](cost-environment-design.md) |

---

## 未解決事項の扱い

各定義書の末尾に「未解決事項」を置き、現時点で決まっていない項目・既知のリスクだけを記載する。解決したら定義書本文に反映して未解決事項から削除する（解決の経緯は活動ログ・ADRに残す）。
