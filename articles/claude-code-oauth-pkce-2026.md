---
title: "Claude CodeでOAuth2 PKCEフローを設計する：認可コード・SPAセキュア認証・リフレッシュ"
emoji: "🔐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "oauth"]
published: true
published_at: "2026-03-13 14:00"
---

## はじめに

SPAでimplicit flowを使うのは危険——PKCE（Proof Key for Code Exchange）で認可コードフローを安全に実装する。Claude Codeにサーバーサイドとクライアントサイドの実装を設計させる。

---

## CLAUDE.mdにOAuth PKCE設計ルールを書く

```markdown
## OAuth2 PKCE設計ルール

### PKCEフロー
- code_verifier: 43-128文字のランダム文字列
- code_challenge: SHA256(code_verifier)のBase64URL
- 認可URLにcode_challenge + code_challenge_method=S256を付与
- トークン交換時にcode_verifierを送信

### トークン管理
- access_token: 15分（短命）
- refresh_token: 30日（回転式）
- access_tokenはメモリ管理（localStorage禁止）
- refresh_tokenはhttpOnly Cookie

### セキュリティ
- state: CSRF防止（32バイトランダム）
- nonce: リプレイ攻撃防止
- PKCE: コード傍受攻撃防止
```

---

## OAuth PKCE実装の生成

```
OAuth2 PKCEフロー認証を設計してください。

要件：
- PKCE（code_verifier/challenge）
- Google/GitHub OAuth対応
- 回転式リフレッシュトークン
- SPA向けセキュアなトークン管理

生成ファイル: src/auth/oauth/
```

---

## 生成されるOAuth PKCE実装

```typescript
// src/auth/oauth/pkceClient.ts — クライアントサイド（SPA）

export class PKCEClient {
  // 認可フローの開始（PKCEパラメータ生成）
  async initiateAuth(provider: 'google' | 'github'): Promise<void> {
    // code_verifier: 96バイト → Base64URL
    const codeVerifier = this.generateCodeVerifier();

    // code_challenge: SHA256(code_verifier) → Base64URL
    const codeChallenge = await this.generateCodeChallenge(codeVerifier);

    // state: CSRF防止（セッションストレージに保存）
    const state = crypto.randomUUID();
    const nonce = crypto.randomUUID();

    sessionStorage.setItem(`pkce_verifier`, codeVerifier);
    sessionStorage.setItem(`pkce_state`, state);
    sessionStorage.setItem(`pkce_nonce`, nonce);

    const authUrl = new URL(PROVIDER_CONFIGS[provider].authEndpoint);
    authUrl.searchParams.set('client_id', PROVIDER_CONFIGS[provider].clientId);
    authUrl.searchParams.set('redirect_uri', `${window.location.origin}/auth/callback`);
    authUrl.searchParams.set('response_type', 'code');
    authUrl.searchParams.set('scope', 'openid email profile');
    authUrl.searchParams.set('state', state);
    authUrl.searchParams.set('nonce', nonce);
    authUrl.searchParams.set('code_challenge', codeChallenge);
    authUrl.searchParams.set('code_challenge_method', 'S256');

    window.location.href = authUrl.toString();
  }

  // コールバック処理
  async handleCallback(searchParams: URLSearchParams): Promise<void> {
    const code = searchParams.get('code');
    const returnedState = searchParams.get('state');
    const error = searchParams.get('error');

    if (error) throw new AuthError(`OAuth error: ${error}`);
    if (!code) throw new AuthError('No authorization code');

    // state検証（CSRF防止）
    const savedState = sessionStorage.getItem('pkce_state');
    if (returnedState !== savedState) throw new AuthError('State mismatch: possible CSRF attack');

    const codeVerifier = sessionStorage.getItem('pkce_verifier');
    sessionStorage.removeItem('pkce_state');
    sessionStorage.removeItem('pkce_verifier');
    sessionStorage.removeItem('pkce_nonce');

    // サーバーにコードとverifierを送信（トークン交換はサーバーサイドで）
    const response = await fetch('/api/auth/oauth/callback', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include', // refresh_token cookieを受け取るため
      body: JSON.stringify({ code, codeVerifier }),
    });

    const { accessToken } = await response.json();
    // access_tokenはメモリに保管（XSS対策のため）
    this.accessToken = accessToken;
  }

  private generateCodeVerifier(): string {
    const array = new Uint8Array(96);
    crypto.getRandomValues(array);
    return btoa(String.fromCharCode(...array))
      .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
  }

  private async generateCodeChallenge(verifier: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const digest = await crypto.subtle.digest('SHA-256', data);
    return btoa(String.fromCharCode(...new Uint8Array(digest)))
      .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
  }
}
```

```typescript
// src/auth/oauth/oauthServer.ts — サーバーサイド（トークン交換）

// OAuthコールバック処理（code → access_token交換）
export async function handleOAuthCallback(
  code: string,
  codeVerifier: string,
  provider: 'google' | 'github'
): Promise<{ accessToken: string }> {
  // Providerからトークンを取得（code + code_verifierで検証）
  const tokenResponse = await fetch(PROVIDER_CONFIGS[provider].tokenEndpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: process.env.OAUTH_REDIRECT_URI!,
      client_id: PROVIDER_CONFIGS[provider].clientId,
      client_secret: PROVIDER_CONFIGS[provider].clientSecret,
      code_verifier: codeVerifier, // PKCEの検証
    }),
  });

  const tokens = await tokenResponse.json();
  const userInfo = await fetchUserInfo(tokens.access_token, provider);

  // ユーザーをUpsert
  const user = await prisma.user.upsert({
    where: { email: userInfo.email },
    create: {
      email: userInfo.email,
      name: userInfo.name,
      avatarUrl: userInfo.picture,
      provider,
      providerId: userInfo.id,
    },
    update: { name: userInfo.name, avatarUrl: userInfo.picture },
  });

  // 自前のJWTを発行
  const accessToken = await generateJWT({ sub: user.id, role: user.role }, { expiresIn: '15m' });
  const refreshToken = generateSecureToken(48); // 48バイトランダム

  // リフレッシュトークンをDBに保存
  await prisma.refreshToken.create({
    data: {
      token: hashToken(refreshToken), // ハッシュして保存
      userId: user.id,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    },
  });

  return { accessToken, refreshToken };
}

// リフレッシュトークンを回転（使用後に新しいトークンを発行）
export async function rotateRefreshToken(
  oldRefreshToken: string
): Promise<{ accessToken: string; refreshToken: string }> {
  const tokenHash = hashToken(oldRefreshToken);
  const stored = await prisma.refreshToken.findFirst({
    where: { token: tokenHash, revokedAt: null, expiresAt: { gt: new Date() } },
  });

  if (!stored) throw new UnauthorizedError('Invalid or expired refresh token');

  // 古いトークンを無効化（回転式）
  await prisma.refreshToken.update({
    where: { id: stored.id },
    data: { revokedAt: new Date() },
  });

  // 新しいトークンペアを発行
  const newAccessToken = await generateJWT({ sub: stored.userId }, { expiresIn: '15m' });
  const newRefreshToken = generateSecureToken(48);

  await prisma.refreshToken.create({
    data: {
      token: hashToken(newRefreshToken),
      userId: stored.userId,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      parentTokenId: stored.id, // リフレッシュトークンチェーンの追跡
    },
  });

  return { accessToken: newAccessToken, refreshToken: newRefreshToken };
}
```

---

## まとめ

Claude CodeでOAuth2 PKCEフローを設計する：

1. **CLAUDE.md** にPKCEフロー・access_token 15分・refresh_token 30日回転式・httpOnly Cookieを明記
2. **code_verifier** をセッションストレージに一時保存し、コールバック後に削除
3. **state検証** でCSRF攻撃を防ぎ、**code_verifier検証** でコード傍受攻撃を防ぐ
4. **回転式refresh_token** で使用後に新しいトークンを発行（リプレイ攻撃防止）

---

*OAuth設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
