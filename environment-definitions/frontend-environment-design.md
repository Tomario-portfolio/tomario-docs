# frontend 環境定義

CloudFront・S3（静的ファイル配信）を管理するコンポーネント（`modules/frontend`）。
独自ドメインは使用せず、CloudFrontのデフォルトドメイン（xxxx.cloudfront.net）とデフォルト証明書でHTTPSを提供する。

---

## S3（静的ファイル配信用）

| 項目 | dev | staging | production |
|------|-----|---------|---|
| バケット名 | tomario-dev-frontend | tomario-staging-frontend | tomario-production-frontend |
| パブリックアクセス | ブロック（CloudFront経由のみ許可） | 同左 | 同左 |
| 暗号化 | SSE-S3（AES256、明示設定） | 同左 | 同左 |
| バージョニング | 無効 | 無効 | 有効 |
| ライフサイクルポリシー | 90日後にIntelligent-Tieringへ移行 | 同左 | 同左 |
| 配信ファイル | HTML / CSS / JavaScript | 同左 | 同左 |

---

## CloudFront

| 項目 | dev | staging | production |
|------|-----|---------|---|
| オリジン（静的） | S3バケット | S3バケット | S3バケット |
| オリジン（動的） | ALB（HTTP、`X-Origin-Verify`ヘッダーを付与） | 同左 | 同左 |
| 証明書 | CloudFrontデフォルト証明書（*.cloudfront.net） | 同左 | 同左 |
| HTTPリダイレクト | HTTP → HTTPS | HTTP → HTTPS | HTTP → HTTPS |
| キャッシュ（静的） | 有効 | 有効 | 有効 |
| キャッシュ（動的 /api/*） | 無効 | 無効 | 無効 |
| WAF | なし | なし | CloudFront用Web ACL（必要な期間のみ。[security-environment-design.md](security-environment-design.md)参照） |

### パスルーティング

| パス | 転送先 | 備考 |
|------|-------|------|
| /api/* | ALB | Flaskアプリへ動的リクエストを転送 |
| /* | S3 | 静的ファイルを配信 |

---

## Route53・ACM

使用しない（[ADR: 独自ドメイン・ACM証明書を導入しない](../../tomario-steering/adr/infra/frontend/001-no-custom-domain.md)参照）。

---

## 未解決事項

なし
