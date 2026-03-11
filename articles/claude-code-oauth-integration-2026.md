---
title: "Claude CodeでOAuth連携を設計する：GitHub/Google ログインとセキュリティ"
emoji: "🔗"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "oauth", "security"]
published: true
---

## はじめに

OAuthのstate検証を省略するとCSRFで乗っ取られる。callback URLをハードコードすると環境切り替えで壊れる。Claude Codeに安全なOAuth実装を生成させる。

---

## CLAUDE.mdにOAuthルールを書く

```markdown
## OAuth設計ルール

### セキュリティ（必須）
- state パラメータを必ず生成・検証（CSRF対策）
- stateはcryptoで生成（推測不能）、Redisに保存（TTL 10分）
- callbackで state が一致しない場合は拒否
- PKCE（code_verifier/code_challenge）を使う（推奨）

### コールバックURL
- callback URLは環境変数から構築（ハードコード禁止）
- 本番/ステージング/開発で自動切り替え

### トークン管理
- OAuthアクセストークンは暗号化してDBに保存
- リフレッシュトークンも保存（期限切れ時に自動更新）
- ユーザーがアカウント連携解除したら即削除
```

---

## OAuthフローの生成

```
GitHub OAuthログインを設計してください。

フロー:
1. GET /auth/github → GitHubの認可ページにリダイレクト（stateとPKCE付き）
2. GET /auth/github/callback → stateを検証、アクセストークンを取得、ユーザー情報を取得
3. 既存ユーザーならJWTを発行、新規ならDB登録後にJWT発行

要件：
- state検証必須
- Redisでstateを管理（TTL 10分）
- ユーザーメールが未公開でも動作
- 既存ユーザーにOAuthアカウントを連携

生成ファイル: src/routes/auth/github.ts
```

---

## 生成されるOAuth実装

```typescript
// src/routes/auth/github.ts
import crypto from 'crypto';
import { Router } from 'express';

const router = Router();
const GITHUB_CLIENT_ID = process.env.GITHUB_CLIENT_ID!;
const GITHUB_CLIENT_SECRET = process.env.GITHUB_CLIENT_SECRET!;
const CALLBACK_URL = `${process.env.APP_URL}/auth/github/callback`;
const STATE_TTL = 600; // 10分

// Step 1: GitHubの認可ページにリダイレクト
router.get('/auth/github', async (req, res) => {
  // stateをcryptoで生成（CSRF対策）
  const state = crypto.randomBytes(32).toString('hex');

  // Redisに保存（TTL 10分）
  await redis.set(`oauth:state:${state}`, '1', { EX: STATE_TTL });

  const params = new URLSearchParams({
    client_id: GITHUB_CLIENT_ID,
    redirect_uri: CALLBACK_URL,
    scope: 'user:email',
    state,
  });

  res.redirect(`https://github.com/login/oauth/authorize?${params}`);
});

// Step 2: GitHubからのコールバック
router.get('/auth/github/callback', async (req, res) => {
  const { code, state } = req.query as { code: string; state: string };

  if (!code || !state) {
    return res.status(400).json({ error: 'Missing code or state' });
  }

  // state検証（CSRF対策）
  const stateKey = `oauth:state:${state}`;
  const storedState = await redis.get(stateKey);

  if (!storedState) {
    return res.status(400).json({ error: 'Invalid or expired state' });
  }

  // 使用済みstateを削除（リプレイ攻撃防止）
  await redis.del(stateKey);

  // GitHubからアクセストークン取得
  const tokenResponse = await fetch('https://github.com/login/oauth/access_token', {
    method: 'POST',
    headers: { 'Accept': 'application/json', 'Content-Type': 'application/json' },
    body: JSON.stringify({
      client_id: GITHUB_CLIENT_ID,
      client_secret: GITHUB_CLIENT_SECRET,
      code,
      redirect_uri: CALLBACK_URL,
    }),
  });

  const { access_token } = await tokenResponse.json();
  if (!access_token) {
    return res.status(400).json({ error: 'Failed to get access token' });
  }

  // GitHubのユーザー情報を取得
  const [userRes, emailRes] = await Promise.all([
    fetch('https://api.github.com/user', {
      headers: { Authorization: `Bearer ${access_token}` },
    }),
    fetch('https://api.github.com/user/emails', {
      headers: { Authorization: `Bearer ${access_token}` },
    }),
  ]);

  const githubUser = await userRes.json();
  const emails = await emailRes.json();
  const primaryEmail = emails.find((e: any) => e.primary)?.email;

  // ユーザーを登録または更新
  const user = await prisma.user.upsert({
    where: { githubId: String(githubUser.id) },
    create: {
      githubId: String(githubUser.id),
      email: primaryEmail ?? `github_${githubUser.id}@noreply.github.com`,
      name: githubUser.name ?? githubUser.login,
      avatarUrl: githubUser.avatar_url,
    },
    update: {
      name: githubUser.name ?? githubUser.login,
      avatarUrl: githubUser.avatar_url,
    },
  });

  // JWTを発行してクライアントにリダイレクト
  const { accessToken } = await generateTokenPair(user.id, user.tenantId ?? 'default');

  // フロントエンドにトークンを渡す（URLパラメータは避けてcookieが望ましい）
  res.cookie('access_token', accessToken, { httpOnly: true, secure: true, maxAge: 15 * 60 * 1000 });
  res.redirect(`${process.env.FRONTEND_URL}/dashboard`);
});

export { router as githubAuthRouter };
```

---

## まとめ

Claude CodeでOAuth連携を設計する：

1. **CLAUDE.md** にstate検証必須・callback URLをenv構築・トークン暗号化保存を明記
2. **state = crypto.randomBytes(32)** でCSRF対策
3. **Redis TTL 10分** でstateを管理（使用後即削除）
4. **upsert** で既存ユーザーへの連携・新規登録を一括処理

---

*OAuth実装のセキュリティレビュー（state検証漏れ、オープンリダイレクト）は **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
