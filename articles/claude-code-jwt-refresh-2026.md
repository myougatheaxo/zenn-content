---
title: "Claude CodeでJWT認証を設計する：リフレッシュトークンとローテーション戦略"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "jwt", "security"]
published: true
---

## はじめに

アクセストークンの有効期限を短くするとセキュリティが上がるが、頻繁にログアウトされてUXが下がる。リフレッシュトークンを正しく実装して両立させる。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにJWT認証ルールを書く

```markdown
## JWT認証設計ルール

### トークン設計
- アクセストークン: 有効期限15分、HTTPヘッダーで送受信
- リフレッシュトークン: 有効期限30日、HttpOnly Cookieで送受信
- リフレッシュトークンローテーション: 使用時に新しいトークンを発行して古いものを無効化

### セキュリティ
- リフレッシュトークンはDBに保存（ハッシュ化）
- 盗難検知: 無効なリフレッシュトークンが使われたらそのユーザーの全セッション破棄
- アクセストークンにrefreshTokenIdを含めてファミリー管理

### Cookie設定
- HttpOnly: true（XSS対策）
- Secure: true（HTTPS必須）
- SameSite: strict（CSRF対策）
- Path: /auth/refresh（リフレッシュ専用パスのみ）
```

---

## JWT認証の生成

```
JWTリフレッシュトークン認証を設計してください。

要件：
- アクセストークン15分 + リフレッシュトークン30日
- リフレッシュトークンはHttpOnly Cookie
- リフレッシュ時にローテーション（旧トークン無効化）
- 盗難検知でファミリー全体を無効化
- リフレッシュトークンはDB保存（ハッシュ化）

生成ファイル: src/services/authService.ts
```

---

## 生成されるJWT認証

```typescript
// src/services/authService.ts
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

const ACCESS_TOKEN_TTL = 15 * 60;       // 15分
const REFRESH_TOKEN_TTL = 30 * 24 * 60 * 60; // 30日

interface AccessTokenPayload {
  userId: string;
  tenantId: string;
  refreshTokenId: string; // ファミリー追跡用
}

export async function generateTokenPair(
  userId: string,
  tenantId: string
): Promise<{ accessToken: string; refreshToken: string; refreshTokenId: string }> {
  const refreshTokenId = crypto.randomUUID();
  const rawRefreshToken = crypto.randomBytes(64).toString('hex');

  // リフレッシュトークンをDBにハッシュ化して保存
  const tokenHash = crypto.createHash('sha256').update(rawRefreshToken).digest('hex');

  await prisma.refreshToken.create({
    data: {
      id: refreshTokenId,
      userId,
      tokenHash,
      expiresAt: new Date(Date.now() + REFRESH_TOKEN_TTL * 1000),
    },
  });

  const accessToken = jwt.sign(
    { userId, tenantId, refreshTokenId } as AccessTokenPayload,
    process.env.JWT_SECRET!,
    { expiresIn: ACCESS_TOKEN_TTL }
  );

  return { accessToken, refreshToken: rawRefreshToken, refreshTokenId };
}

export async function rotateRefreshToken(
  rawRefreshToken: string
): Promise<{ accessToken: string; refreshToken: string } | null> {
  const tokenHash = crypto.createHash('sha256').update(rawRefreshToken).digest('hex');

  const stored = await prisma.refreshToken.findUnique({ where: { tokenHash } });

  if (!stored) {
    // 存在しないトークン = 盗難の可能性
    // → 関連するリフレッシュトークンを特定して全部無効化は困難なため、
    //   このトークンのuserIdが分かればそのユーザーの全セッションを破棄
    logger.warn({ tokenHash }, 'Unknown refresh token used — possible theft');
    return null;
  }

  if (stored.expiresAt < new Date() || stored.revokedAt) {
    // 有効期限切れ or 既に無効化済み（盗難検知）
    if (stored.revokedAt) {
      // 既に使用済みのトークンが再利用された → 盗難
      await prisma.refreshToken.updateMany({
        where: { userId: stored.userId, revokedAt: null },
        data: { revokedAt: new Date() },
      });
      logger.error({ userId: stored.userId }, 'Refresh token reuse detected — all sessions revoked');
    }
    return null;
  }

  // 旧トークンを無効化（ローテーション）
  await prisma.refreshToken.update({
    where: { id: stored.id },
    data: { revokedAt: new Date() },
  });

  // 新しいトークンペアを発行
  const user = await prisma.user.findUnique({
    where: { id: stored.userId },
    select: { id: true, tenantId: true },
  });
  if (!user) return null;

  const { accessToken, refreshToken } = await generateTokenPair(user.id, user.tenantId);
  return { accessToken, refreshToken };
}
```

```typescript
// src/routes/auth.ts
router.post('/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies['refresh_token'];

  if (!refreshToken) {
    return res.status(401).json({ error: 'No refresh token' });
  }

  const result = await rotateRefreshToken(refreshToken);

  if (!result) {
    // 無効なトークン: Cookieを削除してログアウト
    res.clearCookie('refresh_token');
    return res.status(401).json({ error: 'Invalid or expired refresh token' });
  }

  // 新しいリフレッシュトークンをHttpOnly Cookieで設定
  res.cookie('refresh_token', result.refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    path: '/auth/refresh',
    maxAge: 30 * 24 * 60 * 60 * 1000,
  });

  res.json({ accessToken: result.accessToken });
});
```

---

## まとめ

Claude CodeでJWT認証を設計する：

1. **CLAUDE.md** にトークン有効期限・Cookie設定・ローテーションルールを明記
2. **リフレッシュトークンローテーション** で使用済みトークンをDBで無効化
3. **盗難検知** で無効トークン再利用時に全セッション破棄
4. **HttpOnly Cookie** でXSSからリフレッシュトークンを保護

---

*JWT認証の脆弱性（トークン漏洩リスク、ローテーション未実装）を検出するスキルは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
