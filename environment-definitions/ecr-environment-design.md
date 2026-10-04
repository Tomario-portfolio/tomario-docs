# ecr 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

ECR（Elastic Container Registry）を管理する独立コンポーネント。

**新設の経緯（2026-07-10）：** 当初は`modules/backend`内でECS等と一緒に管理していたが、以下の理由で`modules/ecr`として切り出し、`envs/nonprod/shared`（アカウント単位の共通環境）へ移設した。

- ECSタスク定義は既にGraviton/bootstrap_image対応（`var.bootstrap_image`参照）が入っており、`modules/backend`内でECRの出力（`ecr_repository_url`）を実際には使っていなかった
- イメージは「ビルド1回・複数env（dev/staging）にデプロイ」という性質上、workload環境ごとに重複して持つ意味がない

---

## モジュール構成

| 項目 | 内容 |
|------|------|
| モジュールパス | `modules/ecr` |
| 入力変数 | `env`（タグ付け用）、`name`（リポジトリ名を直接指定） |
| 出力 | `repository_url`、`repository_name` |

`name`をenvから自動生成（`tomario-${var.env}-app`）にしなかった理由：ECRリポジトリの`name`属性はAWS側でリネーム不可（変更するとリソース置き換え＝実イメージ消失）。既存の`tomario-app`という名前をそのまま引き継ぐ必要があったため、呼び出し側で名前を明示的に渡す設計にした。

---

## リポジトリ一覧

| 環境 | リポジトリ名 | 管理場所 | 備考 |
|------|------------|---------|------|
| dev / staging | tomario-app | envs/nonprod/shared | 既存名を維持。dev/staging間で共有 |
| production | tomario-production-app | envs/prod/production/ecr | 新規作成。nonprodとはクロスアカウント共有なし |

| 項目 | 内容 |
|------|------|
| イメージタグの変更可否 | 変更不可（IMMUTABLE） |
| スキャン設定 | pushのたびにスキャン |
| ライフサイクルポリシー | 最新5世代のみ保持、古いイメージを自動削除 |

---

## state移行の記録（2026-07-10実施）

既存の`tomario-app`リポジトリ（実イメージ入り）を、destroy→createではなく`terraform state mv`で無停止移行した。

1. `terraform state pull`でdev/shared双方のstateをローカルにバックアップ
2. `terraform state mv -state=<dev状態> -state-out=<shared状態>`で`module.backend.aws_ecr_repository.this`等を`module.ecr.aws_ecr_repository.this`へ移動
3. `terraform state push`で両env分を反映
4. 両envで`terraform plan`を実行し、リポジトリ本体（`name`属性）に変更が無く、タグ更新のみであることを確認（実害なし）
5. AWS CLIで`createdAt`が移行前と同一であること、既存イメージが全て残っていることを確認

このため、移行前後でイメージの再pushや再ビルドは一切発生していない。

---

## bootstrap_image

ECSタスク定義の初期イメージ（`var.bootstrap_image`）は本コンポーネントとは別リソースだが、密接に関連する。当初パブリックのECR Gallery（`public.ecr.aws/docker/library/nginx:latest`）を参照していたが、NAT Gatewayの無いVPC構成では到達できず、サービス再作成のたびにクラッシュループを起こす問題があった（2026-07-10発覚）。`tomario-app`（本コンポーネントで管理するプライベートECR）内に`bootstrap`タグでプレースホルダーイメージを一度だけpushし、そちらを参照する形に修正した。詳細は`compute-environment-design.md`相当（`backend-environment-design.md`）参照。

---

## staging→production 昇格フロー（production初回構築に必須）

productionは別リポジトリ（クロスアカウント共有なし）のため、staging→production昇格時は再ビルドせず、以下の手順でイメージを引き継ぐ。production環境が起動するにはイメージの引き継ぎ自体が必須のため、Blue/Green等のWave B項目とは異なりproduction初回構築（Wave A）の一部として実装する。

1. tomario-appのデプロイパイプラインで、stagingデプロイ後に手動承認ステップを設ける
2. 承認後、stagingで検証済みのイメージをdigest指定でpull
3. re-tagして`tomario-production-app`へpush（再ビルドはしない）
4. productionのECSタスク定義を、そのdigestのイメージで更新

---

## TBD解消事項

| 項目 | 内容 |
|------|------|
| production用ECRリポジトリの作成 | production環境構築時に対応 |
| promoteジョブの実装 | tomario-app側のCI/CDに追加 |
