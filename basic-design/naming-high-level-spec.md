# 命名規則

## 目的

命名規則を定める目的は以下の通り。

- リソースの役割を示す
- 対象リソースの取り間違えを防ぐ
- 管理上の識別性を高める

## 基本ルール

形式：`{システム名}-{環境}-{リソース種別}` で統一する。
単語間はハイフン（-）で結び、英小文字と数字のみを使用する。
マルチバイト文字・英大文字・アンダースコアは使用しない。

環境（`{環境}`に入る値）は `dev` / `staging` / `shared` / `production` の4種類とする。
`shared`はnonprodアカウント共通のGuardDuty・CloudTrail・Budgets・ECR等、アカウント単位のシングルトンリソース専用の環境名で、workload環境（dev/staging/production）とは別ディレクトリ・別stateで管理する。

ECRリポジトリ（`tomario-app`）のみ例外で、既存リポジトリ名をそのまま維持する（AWS側でリネーム不可のため。dev/stagingで共有し、productionは`tomario-production-app`という別リポジトリを使用）。

リソース別の具体的な名称は[環境定義書](../environment-definitions/)に記載する。

## タグ規則

すべてのリソースに以下のタグを付与する。
環境・プロジェクト単位でのリソース検索やコスト把握を容易にするため。

| タグキー | タグ値 |
|---------|--------|
| Name | 命名規則で定めた名称 |
| Project | tomario |
| Environment | dev / staging / shared / production |
| ManagedBy | terraform |
