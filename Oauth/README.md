# OAuth / 認証・認可

OAuth 2.0 および認証・認可に関する学習資料をまとめています。

## 📚 コンテンツ

### [OAuth 2.0の仕組み](./oauth-explanation.md)

OAuth 2.0の基本的な認可フローを詳しく解説しています。

**内容:**
- OAuth 2.0の主要な登場人物（Resource Owner、Client、Authorization Server、Resource Server）
- 認可コードフロー（Authorization Code Flow）の詳細
- 各ステップで送受信されるデータの詳細
  - 認可リクエスト
  - 認可コード発行
  - アクセストークン取得
  - リソースアクセス
  - トークン更新
- セキュリティポイント（state、client_secret、HTTPS、有効期限など）
- その他のOAuthフロー（Implicit Flow、Client Credentials Flow、PKCE）

**図解:**
- マーメイドシーケンス図で全フローを視覚化
- サーバー間でやり取りされるデータの中身を詳細に記載

---

### [ハイブリッド認証戦略](./oauth-hybrid-authentication.md)

外部OAuthプロバイダーでの初回認証後、独自トークンで管理する実践的な実装パターンです。

**内容:**
- ハイブリッド認証の概要とメリット
  - 認証の信頼性とパフォーマンスの両立
  - 独自の権限管理
  - オフライン対応
- 詳細なフロー図
  - 初回ログイン（外部OAuth認証）
  - 2回目以降のアクセス（独自トークン使用）
  - トークン更新
  - ログアウト
- データ構造の詳細
  - 外部認証コールバック
  - JWTペイロード
  - トークン更新リクエスト/レスポンス
- データベーススキーマ例
  - usersテーブル
  - sessionsテーブル
  - external_auth_tokensテーブル
- セキュリティ考慮事項
  - トークンの安全な保存方法
  - 有効期限設定
  - JWT検証フロー
  - CSRF対策
- 実装例（Node.js/Express）

**図解:**
- マーメイドシーケンス図で全フローを視覚化
- データベース間の連携も含めた詳細な図

---

## 🎯 学習のポイント

### OAuth 2.0の基礎を理解する
1. まず [OAuth 2.0の仕組み](./oauth-explanation.md) で基本的な認可フローを理解
2. 各ステップでやり取りされるデータを把握
3. セキュリティポイントを確認

### 実践的な実装パターンを学ぶ
1. [ハイブリッド認証戦略](./oauth-hybrid-authentication.md) で実務的な実装方法を学習
2. データベース設計とセッション管理の方法を理解
3. セキュリティベストプラクティスを習得

## 🔐 セキュリティのベストプラクティス

### 必ず守るべきこと
- ✅ すべての通信をHTTPSで行う
- ✅ client_secretは絶対に公開しない（サーバーサイドのみ）
- ✅ stateパラメータでCSRF攻撃を防ぐ
- ✅ トークンは適切な有効期限を設定
- ✅ トークンはハッシュ化してDB保存
- ✅ XSS対策（LocalStorageにAccessTokenを保存しない）

### 推奨事項
- 🔒 PKCE（Proof Key for Code Exchange）の使用（モバイル/SPA）
- 🔒 Refresh Tokenのローテーション
- 🔒 トークンの定期的な更新
- 🔒 セッション管理とログ記録

## 🚀 実装時の検討事項

### 外部OAuthプロバイダーの選択
- **Google OAuth**: 広く普及、信頼性が高い
- **GitHub OAuth**: 開発者向けサービスに最適
- **Auth0**: エンタープライズ向け、多機能
- **Firebase Authentication**: Googleのマネージドサービス

### トークン管理戦略
1. **完全に外部依存**: シンプルだが柔軟性に欠ける
2. **ハイブリッド**: 初回のみ外部認証、以降は独自トークン（推奨）
3. **完全に独自実装**: 最大の柔軟性だが実装コストが高い

## 📖 参考リソース

### 公式仕様
- [OAuth 2.0 RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OAuth 2.0 for Browser-Based Apps](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps)

### プロバイダー別ドキュメント
- [Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)
- [GitHub OAuth Apps](https://docs.github.com/en/apps/oauth-apps)
- [Auth0 Documentation](https://auth0.com/docs)

---

**最終更新: 2024-11-15**
