# frontend 環境定義

## 基本方針

| 項目 | 内容 |
|------|------|
| Terraformバージョン | ~> 1.5 |
| AWSプロバイダー | hashicorp/aws ~> 6.0 |

CloudFront・S3を管理するコンポーネント。
他のコンポーネント（network・backend）と独立して構築できる。
独自ドメインは現時点では使用しない。CloudFrontのデフォルトドメイン（xxxx.cloudfront.net）でアクセスする。

---

## S3（静的ファイル配信用）

| 項目 | dev | staging | production |
|------|-----|---------|---|
| バケット名 | tomario-dev-frontend | tomario-staging-frontend | tomario-production-frontend |
| パブリックアクセス | ブロック（CloudFront経由のみ許可） | ブロック（CloudFront経由のみ許可） | ブロック（CloudFront経由のみ許可） |
| バージョニング | 無効 | 無効 | **有効**（誤って上書き・削除したデプロイ資産を復元できるようにする。静的ファイルのみでサイズが小さく追加コストは軽微） |
| ライフサイクルポリシー | 90日後にIntelligent-Tieringへ移行 | 同左 | 同左 |
| 配信ファイル | HTML / CSS / JavaScript | HTML / CSS / JavaScript | HTML / CSS / JavaScript |

---

## ACM証明書・独自ドメイン（検討中・後回し）

現時点では不使用。CloudFrontのデフォルト証明書（*.cloudfront.net）でHTTPSを提供する。
独自ドメイン取得は技術的な必要性が薄く、他設計への影響もないためいつでも後付け可能と判断し、優先度を下げて「検討中・後回し」としている（2026-07-11）。ACM証明書は独自ドメインとセットのため、ドメインを取得しない限り不要。

---

## CloudFront

| 項目 | dev | staging | production |
|------|-----|---------|---|
| オリジン①（静的） | S3バケット | S3バケット | S3バケット |
| オリジン②（動的） | ALB | ALB | ALB |
| HTTPS | 有効（CloudFrontデフォルト証明書） | 有効（CloudFrontデフォルト証明書） | 有効（CloudFrontデフォルト証明書） |
| HTTPリダイレクト | HTTP → HTTPS | HTTP → HTTPS | HTTP → HTTPS |
| キャッシュ（静的） | 有効 | 有効 | 有効 |
| キャッシュ（動的 /api/*） | 無効 | 無効 | 無効 |
| WAF | なし | なし | あり（CloudFront用Web ACL。常時起動はせず面接期間のみ作成、詳細は`cost-environment-design.md`・`security-environment-design.md`参照） |

### パスルーティング

| パス | 転送先 | 備考 |
|------|-------|------|
| /api/* | ALB | Flaskアプリへ動的リクエストを転送 |
| /* | S3 | 静的ファイルを配信 |

---

## Route53

現時点では不使用。独自ドメインと同様「検討中・後回し」（上記参照）。

---

## TBD解消事項

なし
