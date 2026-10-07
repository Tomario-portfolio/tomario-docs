# ecr 環境定義

ECR（Elastic Container Registry）を管理するコンポーネント（`modules/ecr`）。
イメージは「1回ビルドして複数環境にデプロイする」ものなので、workload環境ごとではなくアカウントごとにリポジトリを持つ。

---

## モジュール構成

| 項目 | 内容 |
|------|------|
| モジュールパス | `modules/ecr` |
| 入力変数 | `env`（タグ付け用）、`name`（リポジトリ名を直接指定） |
| 出力 | `repository_url`、`repository_name` |

リポジトリ名は`env`から自動生成せず、呼び出し側で明示的に渡す（ECRのリポジトリ名はリネーム不可で、既存名`tomario-app`を維持する必要があるため）。

---

## リポジトリ

| 項目 | nonprod（dev / staging共有） | production |
|------|------------|---------|
| リポジトリ名 | tomario-app | tomario-production-app |
| 管理場所 | envs/nonprod/shared | envs/prod/production/ecr |
| イメージタグの変更可否 | IMMUTABLE | IMMUTABLE |
| スキャン設定 | push時にスキャン | push時にスキャン |
| ライフサイクルポリシー | 優先度1：`bootstrap`タグを保護（削除対象外）／優先度2：最新5世代のみ保持（`imageCountMoreThan 5`で古いものを削除） | 同左 |
| クロスアカウント共有 | なし | なし |

### bootstrapイメージ

各リポジトリに`bootstrap`タグで実アプリのイメージを置き、ECSタスク定義の初期イメージ（`var.bootstrap_image`）として参照する（[backend-environment-design.md](backend-environment-design.md)参照）。経緯は[ADR: bootstrap_imageのプライベートECR参照化](../../tomario-steering/adr/infra/network/002-bootstrap-image-private-ecr.md)を参照。

---

## staging→production 昇格

productionはnonprodとリポジトリを共有しないため、`tomario-app`の`deploy.yml`の`promote-to-production`ジョブでイメージを引き継ぐ。

1. stagingデプロイ後、手動承認を経てジョブを実行する
2. stagingで検証済みのイメージをdigest指定でpullする
3. re-tagして`tomario-production-app`へpushする（再ビルドはしない）
4. productionのECSタスク定義をそのイメージで更新する（productionのECSサービスが停止中の場合はスキップ）

---

## 未解決事項

なし
