# OAuthの仕組み

## OAuth 2.0の基本的な流れ

OAuth 2.0は、サードパーティアプリケーションがユーザーのリソースに安全にアクセスするための認可フレームワークです。

## 主要な登場人物

1. **Resource Owner（リソース所有者）**: ユーザー
2. **Client（クライアント）**: リソースにアクセスしたいアプリケーション
3. **Authorization Server（認可サーバー）**: 認証・認可を行うサーバー
4. **Resource Server（リソースサーバー）**: 保護されたリソースを持つサーバー

## 認可コードフロー（Authorization Code Flow）

最も一般的で安全なフローです：

```mermaid
sequenceDiagram
    participant User as ユーザー<br/>(Resource Owner)
    participant Client as クライアント<br/>アプリ
    participant AuthServer as 認可サーバー<br/>(Authorization Server)
    participant ResourceServer as リソースサーバー<br/>(Resource Server)

    Note over User,ResourceServer: 1. 認可リクエスト
    User->>Client: アプリ利用開始
    Client->>User: 認可ページへリダイレクト
    activate User
    User->>AuthServer: GET /authorize<br/>?response_type=code<br/>&client_id=CLIENT_ID<br/>&redirect_uri=CALLBACK_URL<br/>&scope=read write<br/>&state=RANDOM_STRING
    deactivate User
    
    Note over User,AuthServer: 2. ユーザー認証と同意
    AuthServer->>User: ログイン画面表示
    User->>AuthServer: ログイン情報送信<br/>(username, password)
    AuthServer->>User: 権限同意画面表示<br/>(scopeの内容)
    User->>AuthServer: 同意ボタンクリック
    
    Note over AuthServer,Client: 3. 認可コード発行
    AuthServer->>Client: リダイレクト<br/>GET CALLBACK_URL<br/>?code=AUTHORIZATION_CODE<br/>&state=RANDOM_STRING
    
    Note over Client,AuthServer: 4. アクセストークン取得
    Client->>AuthServer: POST /token<br/>grant_type=authorization_code<br/>&code=AUTHORIZATION_CODE<br/>&redirect_uri=CALLBACK_URL<br/>&client_id=CLIENT_ID<br/>&client_secret=CLIENT_SECRET
    
    AuthServer->>AuthServer: 検証処理
    AuthServer->>Client: JSON Response<br/>{<br/>  "access_token": "ACCESS_TOKEN",<br/>  "token_type": "Bearer",<br/>  "expires_in": 3600,<br/>  "refresh_token": "REFRESH_TOKEN",<br/>  "scope": "read write"<br/>}
    
    Note over Client,ResourceServer: 5. リソースアクセス
    Client->>ResourceServer: GET /api/user/profile<br/>Header: Authorization: Bearer ACCESS_TOKEN
    ResourceServer->>AuthServer: トークン検証（必要に応じて）
    AuthServer->>ResourceServer: トークン有効
    ResourceServer->>Client: JSON Response<br/>{<br/>  "id": "123",<br/>  "name": "User Name",<br/>  "email": "user@example.com"<br/>}
    
    Note over Client,AuthServer: 6. トークン更新（期限切れ時）
    Client->>AuthServer: POST /token<br/>grant_type=refresh_token<br/>&refresh_token=REFRESH_TOKEN<br/>&client_id=CLIENT_ID<br/>&client_secret=CLIENT_SECRET
    AuthServer->>Client: JSON Response<br/>{<br/>  "access_token": "NEW_ACCESS_TOKEN",<br/>  "token_type": "Bearer",<br/>  "expires_in": 3600<br/>}
```

## 各ステップの詳細データ

### 1. 認可リクエスト（Authorization Request）

```
GET /authorize?
  response_type=code          # 認可コードフローを指定
  &client_id=abc123           # クライアントID
  &redirect_uri=https://client.example.com/callback
  &scope=read write delete    # 要求する権限
  &state=xyz789              # CSRF対策用ランダム文字列
```

**パラメータ説明:**
- `response_type`: 認可フローのタイプ（code = 認可コードフロー）
- `client_id`: 認可サーバーに事前登録したクライアントの識別子
- `redirect_uri`: 認可後にリダイレクトされるコールバックURL
- `scope`: アクセス権限の範囲（read, write, deleteなど）
- `state`: CSRF攻撃を防ぐためのランダムな文字列

### 2. 認可コード発行（Authorization Code）

```
GET https://client.example.com/callback?
  code=SplxlOBeZQQYbYS6WxSbIA    # 認可コード（一時的、数分で期限切れ）
  &state=xyz789                   # 送信したstateと同じ値
```

**データ説明:**
- `code`: 一時的な認可コード（通常5〜10分で有効期限切れ）
- `state`: リクエスト時に送信した値と同じであることを確認

### 3. アクセストークン取得（Token Request）

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://client.example.com/callback
&client_id=abc123
&client_secret=secret456        # クライアントシークレット
```

**パラメータ説明:**
- `grant_type`: トークン取得方法（authorization_code）
- `code`: 前のステップで取得した認可コード
- `redirect_uri`: 認可リクエスト時と同じコールバックURL
- `client_id`: クライアントID
- `client_secret`: クライアントシークレット（サーバーサイドでのみ使用）

**レスポンス:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "scope": "read write"
}
```

**レスポンスデータ説明:**
- `access_token`: リソースにアクセスするためのトークン
- `token_type`: トークンのタイプ（通常は "Bearer"）
- `expires_in`: アクセストークンの有効期限（秒単位）
- `refresh_token`: アクセストークンを更新するためのトークン
- `scope`: 実際に付与された権限

### 4. リソースアクセス（API Request）

```
GET /api/user/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**ヘッダー説明:**
- `Authorization`: "Bearer " + アクセストークン

**レスポンス例:**
```json
{
  "id": "123",
  "name": "User Name",
  "email": "user@example.com",
  "created_at": "2024-01-01T00:00:00Z"
}
```

### 5. トークン更新（Refresh Token Request）

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&client_id=abc123
&client_secret=secret456
```

**パラメータ説明:**
- `grant_type`: refresh_token
- `refresh_token`: 前回取得したリフレッシュトークン
- `client_id`: クライアントID
- `client_secret`: クライアントシークレット

**レスポンス:**
```json
{
  "access_token": "NEW_ACCESS_TOKEN_STRING",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "NEW_REFRESH_TOKEN"
}
```

## セキュリティポイント

### 1. state パラメータ
- **目的**: CSRF（Cross-Site Request Forgery）攻撃を防ぐ
- **実装**: ランダムな文字列を生成し、リクエスト時とコールバック時で一致を確認

### 2. client_secret
- **重要**: サーバーサイドでのみ使用、絶対に公開しない
- **保存**: 環境変数や秘密管理システムで安全に管理

### 3. HTTPS必須
- すべてのOAuth通信は必ずHTTPSで暗号化
- 中間者攻撃を防ぐために必須

### 4. 認可コードの有効期限
- 通常: 5〜10分程度
- 一度使用したら無効化

### 5. アクセストークンの有効期限
- 通常: 数時間〜数日
- 短い有効期限で定期的に更新

### 6. Refresh Token
- 長期間有効（数週間〜数ヶ月）
- 安全に保管し、定期的にローテーション

## その他のOAuth 2.0フロー

### Implicit Flow（暗黙的フロー）
- SPA（Single Page Application）向け
- 認可コードを使わず、直接アクセストークンを取得
- **非推奨**: セキュリティ上の理由から、現在はPKCE付き認可コードフローが推奨

### Client Credentials Flow（クライアント認証情報フロー）
- サーバー間通信向け
- ユーザー認証なし、クライアント自身の認証のみ

### Resource Owner Password Credentials Flow
- ユーザー名とパスワードを直接使用
- **非推奨**: OAuth本来の目的に反するため、特別な理由がない限り使用しない

### PKCE（Proof Key for Code Exchange）
- モバイルアプリやSPA向けの拡張
- client_secretを使わずに認可コードフローを安全に実行
- code_verifierとcode_challengeを使用

## まとめ

OAuth 2.0により、クライアントアプリケーションはユーザーのパスワードを知ることなく、限定的な権限でリソースにアクセスできます。

**主な利点:**
- パスワードの共有不要
- 権限の細かい制御（scope）
- トークンの取り消しが可能
- セキュアな認可メカニズム
