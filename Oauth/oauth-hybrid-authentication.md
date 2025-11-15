# ハイブリッド認証戦略：外部OAuth + 独自トークン管理

## 概要

この戦略は実際に多くのアプリケーションで採用されています。初回ログイン時のみ外部の認証サーバー（Google, GitHub, Auth0など）で認証を行い、その後はクライアントアプリ（自社のバックエンド）が独自に発行したトークンを使用する方式です。

## メリット

1. **認証の信頼性**: 初回認証は実績のある外部サービスに任せる
2. **パフォーマンス**: 毎回外部サーバーに問い合わせる必要がない
3. **独自の権限管理**: 自社アプリケーション固有の権限やロールを管理できる
4. **オフライン対応**: 外部サーバーがダウンしても既存ユーザーは利用可能
5. **カスタマイズ性**: トークンの有効期限や更新ポリシーを自由に設定

## フロー図

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Frontend as フロントエンド<br/>(SPA/Mobile)
    participant Backend as バックエンド<br/>(自社APIサーバー)
    participant ExternalAuth as 外部認証サーバー<br/>(Google/GitHub等)
    participant Database as データベース

    Note over User,Database: 【初回ログイン】外部OAuth認証
    
    User->>Frontend: ログインボタンクリック
    Frontend->>ExternalAuth: GET /authorize<br/>?response_type=code<br/>&client_id=YOUR_CLIENT_ID<br/>&redirect_uri=https://yourapp.com/callback<br/>&scope=openid profile email
    
    ExternalAuth->>User: ログイン画面表示
    User->>ExternalAuth: 認証情報入力・同意
    
    ExternalAuth->>Frontend: リダイレクト<br/>https://yourapp.com/callback<br/>?code=EXTERNAL_AUTH_CODE
    
    Frontend->>Backend: POST /auth/external/callback<br/>{<br/>  "code": "EXTERNAL_AUTH_CODE",<br/>  "provider": "google"<br/>}
    
    Backend->>ExternalAuth: POST /token<br/>grant_type=authorization_code<br/>&code=EXTERNAL_AUTH_CODE<br/>&client_id=YOUR_CLIENT_ID<br/>&client_secret=YOUR_SECRET
    
    ExternalAuth->>Backend: {<br/>  "access_token": "ext_token_xxx",<br/>  "id_token": "eyJhbG...",<br/>  "refresh_token": "ext_refresh_yyy"<br/>}
    
    Backend->>ExternalAuth: GET /userinfo<br/>Authorization: Bearer ext_token_xxx
    
    ExternalAuth->>Backend: {<br/>  "sub": "google_12345",<br/>  "email": "user@example.com",<br/>  "name": "User Name",<br/>  "picture": "https://..."<br/>}
    
    Backend->>Database: ユーザー検索/作成<br/>SELECT * FROM users<br/>WHERE provider='google'<br/>AND external_id='google_12345'
    
    Database->>Backend: ユーザー情報
    
    Backend->>Backend: 独自トークン生成<br/>JWT生成（秘密鍵で署名）<br/>有効期限: 7日間
    
    Backend->>Database: セッション保存<br/>INSERT INTO sessions<br/>(user_id, token_hash,<br/>expires_at, refresh_token_hash)
    
    Backend->>Frontend: {<br/>  "access_token": "your_app_token_abc123",<br/>  "refresh_token": "your_app_refresh_xyz789",<br/>  "expires_in": 604800,<br/>  "user": {<br/>    "id": "user_001",<br/>    "email": "user@example.com",<br/>    "name": "User Name",<br/>    "roles": ["user", "premium"]<br/>  }<br/>}
    
    Frontend->>Frontend: トークンをローカルストレージ/SecureStorageに保存
    
    Note over User,Database: 【2回目以降のアクセス】独自トークン使用
    
    User->>Frontend: アプリ起動・ページ遷移
    Frontend->>Backend: GET /api/user/profile<br/>Authorization: Bearer your_app_token_abc123
    
    Backend->>Backend: トークン検証<br/>1. JWT署名検証<br/>2. 有効期限チェック<br/>3. セッション存在確認
    
    Backend->>Database: SELECT * FROM sessions<br/>WHERE token_hash=hash('your_app_token_abc123')<br/>AND expires_at > NOW()
    
    Database->>Backend: セッション情報
    
    Backend->>Frontend: {<br/>  "id": "user_001",<br/>  "email": "user@example.com",<br/>  "name": "User Name",<br/>  "roles": ["user", "premium"],<br/>  "preferences": {...}<br/>}
    
    Note over User,Database: 【トークン更新】有効期限切れ前
    
    Frontend->>Backend: POST /auth/refresh<br/>{<br/>  "refresh_token": "your_app_refresh_xyz789"<br/>}
    
    Backend->>Database: SELECT * FROM sessions<br/>WHERE refresh_token_hash=hash('your_app_refresh_xyz789')<br/>AND refresh_expires_at > NOW()
    
    Database->>Backend: セッション情報
    
    Backend->>Backend: 新しいトークン生成
    
    Backend->>Database: UPDATE sessions<br/>SET token_hash=...,<br/>refresh_token_hash=...,<br/>expires_at=...,<br/>updated_at=NOW()
    
    Backend->>Frontend: {<br/>  "access_token": "new_token_def456",<br/>  "refresh_token": "new_refresh_uvw012",<br/>  "expires_in": 604800<br/>}
    
    Note over User,Database: 【ログアウト】
    
    User->>Frontend: ログアウトボタンクリック
    Frontend->>Backend: POST /auth/logout<br/>Authorization: Bearer your_app_token_abc123
    
    Backend->>Database: DELETE FROM sessions<br/>WHERE token_hash=hash('your_app_token_abc123')
    
    Backend->>Frontend: {"success": true}
    Frontend->>Frontend: ローカルのトークン削除
```

## データ構造の詳細

### 1. 初回ログイン - 外部認証コールバック

**リクエスト:**
```http
POST /auth/external/callback
Content-Type: application/json

{
  "code": "4/0AY0e-g7xxxxxxxxxxx",
  "provider": "google",
  "redirect_uri": "https://yourapp.com/callback"
}
```

**外部サーバーからのユーザー情報取得:**
```json
{
  "sub": "google_1234567890",
  "email": "user@example.com",
  "email_verified": true,
  "name": "田中太郎",
  "given_name": "太郎",
  "family_name": "田中",
  "picture": "https://lh3.googleusercontent.com/a/xxxxx",
  "locale": "ja"
}
```

**自社バックエンドからのレスポンス:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoidXNlcl8wMDEiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20iLCJyb2xlcyI6WyJ1c2VyIl0sImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwNjA0ODAwfQ.xxxxx",
  "refresh_token": "rt_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "token_type": "Bearer",
  "expires_in": 604800,
  "user": {
    "id": "user_001",
    "email": "user@example.com",
    "name": "田中太郎",
    "picture": "https://lh3.googleusercontent.com/a/xxxxx",
    "roles": ["user"],
    "created_at": "2024-01-01T00:00:00Z",
    "last_login": "2024-11-15T06:40:00Z"
  }
}
```

### 2. 2回目以降 - 独自トークンでのAPIアクセス

**リクエスト:**
```http
GET /api/user/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoidXNlcl8wMDEi...
```

**JWTペイロード（デコード後）:**
```json
{
  "user_id": "user_001",
  "email": "user@example.com",
  "roles": ["user", "premium"],
  "session_id": "session_abc123",
  "iat": 1700000000,
  "exp": 1700604800,
  "iss": "https://yourapp.com",
  "aud": "yourapp-api"
}
```

### 3. トークン更新

**リクエスト:**
```http
POST /auth/refresh
Content-Type: application/json

{
  "refresh_token": "rt_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
}
```

**レスポンス:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.new_token_payload.signature",
  "refresh_token": "rt_z9y8x7w6v5u4t3s2r1q0p9o8n7m6l5k4",
  "token_type": "Bearer",
  "expires_in": 604800
}
```

## データベーススキーマ例

### usersテーブル
```sql
CREATE TABLE users (
    id VARCHAR(36) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    picture_url TEXT,
    provider VARCHAR(50) NOT NULL,  -- 'google', 'github', 'facebook'
    external_id VARCHAR(255) NOT NULL,  -- 外部サービスのユーザーID
    roles JSON,  -- ["user", "admin", "premium"]
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    UNIQUE KEY unique_provider_external_id (provider, external_id)
);
```

### sessionsテーブル
```sql
CREATE TABLE sessions (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    token_hash VARCHAR(64) NOT NULL,  -- SHA-256ハッシュ
    refresh_token_hash VARCHAR(64) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    refresh_expires_at TIMESTAMP NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_token_hash (token_hash),
    INDEX idx_refresh_token_hash (refresh_token_hash),
    INDEX idx_user_expires (user_id, expires_at)
);
```

### external_auth_tokensテーブル（オプション）
```sql
CREATE TABLE external_auth_tokens (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    provider VARCHAR(50) NOT NULL,
    access_token TEXT,  -- 暗号化推奨
    refresh_token TEXT,  -- 暗号化推奨
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    UNIQUE KEY unique_user_provider (user_id, provider)
);
```

## セキュリティ考慮事項

### 1. トークンの保存

**フロントエンド:**
- **Webアプリ**: HttpOnly CookieまたはメモリにのみAccessToken保存
- **モバイルアプリ**: Secure Storage (iOS KeyChain, Android KeyStore)
- **絶対に避ける**: LocalStorageへのAccessToken保存（XSS攻撃のリスク）

**バックエンド:**
- トークンは必ずハッシュ化してDB保存
- 外部サービスのトークンは暗号化して保存

### 2. トークンの有効期限設定例

```javascript
const TOKEN_CONFIG = {
  accessToken: {
    expiresIn: '7d',        // アクセストークン: 7日
    algorithm: 'HS256'      // または RS256（公開鍵暗号方式）
  },
  refreshToken: {
    expiresIn: '30d',       // リフレッシュトークン: 30日
    rotationEnabled: true   // 更新時に新しいトークンを発行
  }
};
```

### 3. JWT検証フロー

```javascript
// バックエンドでの検証処理
async function verifyToken(token) {
  // 1. JWT署名検証
  const decoded = jwt.verify(token, SECRET_KEY);
  
  // 2. 有効期限チェック（jwt.verifyで自動的に行われる）
  
  // 3. セッション存在確認
  const tokenHash = crypto.createHash('sha256').update(token).digest('hex');
  const session = await db.query(
    'SELECT * FROM sessions WHERE token_hash = ? AND expires_at > NOW()',
    [tokenHash]
  );
  
  if (!session) {
    throw new Error('Invalid or expired session');
  }
  
  // 4. ユーザー情報取得
  const user = await db.query('SELECT * FROM users WHERE id = ?', [decoded.user_id]);
  
  return { user, session };
}
```

### 4. CSRF対策

```javascript
// SameSite Cookie設定
res.cookie('refresh_token', refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 30 * 24 * 60 * 60 * 1000  // 30日
});
```

## 実装例（バックエンド - Node.js/Express）

```javascript
// 初回ログイン - 外部認証コールバック
app.post('/auth/external/callback', async (req, res) => {
  const { code, provider } = req.body;
  
  // 1. 外部サーバーからアクセストークン取得
  const externalTokens = await exchangeCodeForToken(provider, code);
  
  // 2. ユーザー情報取得
  const externalUser = await fetchUserInfo(provider, externalTokens.access_token);
  
  // 3. ユーザー作成または取得
  let user = await db.users.findOne({
    provider,
    external_id: externalUser.sub
  });
  
  if (!user) {
    user = await db.users.create({
      id: generateUUID(),
      email: externalUser.email,
      name: externalUser.name,
      picture_url: externalUser.picture,
      provider,
      external_id: externalUser.sub,
      roles: ['user']
    });
  }
  
  // 4. 独自トークン生成
  const accessToken = jwt.sign(
    {
      user_id: user.id,
      email: user.email,
      roles: user.roles
    },
    SECRET_KEY,
    { expiresIn: '7d' }
  );
  
  const refreshToken = generateSecureRandomToken();
  
  // 5. セッション保存
  await db.sessions.create({
    id: generateUUID(),
    user_id: user.id,
    token_hash: hashToken(accessToken),
    refresh_token_hash: hashToken(refreshToken),
    expires_at: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    refresh_expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    ip_address: req.ip,
    user_agent: req.headers['user-agent']
  });
  
  // 6. レスポンス
  res.json({
    access_token: accessToken,
    refresh_token: refreshToken,
    token_type: 'Bearer',
    expires_in: 604800,
    user: {
      id: user.id,
      email: user.email,
      name: user.name,
      picture: user.picture_url,
      roles: user.roles
    }
  });
});

// API認証ミドルウェア
app.use('/api/*', async (req, res, next) => {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) throw new Error('No token provided');
    
    const { user, session } = await verifyToken(token);
    req.user = user;
    req.session = session;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Unauthorized' });
  }
});
```

## まとめ

この戦略は以下のような場合に特に有効です：

✅ **採用すべきケース:**
- 独自の権限管理やビジネスロジックが必要
- 高いパフォーマンスが求められる
- 外部サービスのAPI制限を避けたい
- 複数の外部認証プロバイダーをサポート

❌ **注意が必要なケース:**
- シンプルな認証のみで十分な場合
- セッション管理の複雑さを避けたい場合
- 完全にステートレスなシステムを構築したい場合

この方式により、外部認証の信頼性と自社システムの柔軟性の両方を実現できます。
