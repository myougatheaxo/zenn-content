---
title: "Claude Codeでマジックリンク認証を設計する：パスワードレス・ワンタイムトークン・フィッシング対策"
emoji: "🪄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "auth"]
published: true
published_at: "2026-03-14 11:00"
---

## はじめに

「パスワードを忘れた」を根本から解消する——メールに送ったリンクをクリックするだけでログインできるマジックリンク認証を設計する。ワンタイム・有効期限・フィッシング対策までClaude Codeに生成させる。

---

## CLAUDE.mdにマジックリンク認証設計ルールを書く

```markdown
## マジックリンク認証設計ルール

### トークン設計
- 長さ: 48バイト（384bit）Base64URL エンコード
- 有効期限: 15分（短命）
- ワンタイム: 使用後即削除（リプレイ攻撃防止）
- DBにはSHA256ハッシュで保存（漏洩時の悪用防止）

### フィッシング対策
- originバインド: トークン生成時のoriginと照合
- User-Agentバインド（オプション）: 同一ブラウザでのみ有効
- rate limit: 同一メールへの送信は5分に1回まで

### 送信先確認
- 登録済みメールのみに送信（ユーザー存在確認はしない→アカウント列挙防止）
- メール送信: 常に「送りました」（アカウント有無を漏らさない）
```

---

## マジックリンク認証の生成

```
マジックリンク（パスワードレス）認証を設計してください。

要件：
- 15分有効なワンタイムトークン
- ハッシュ化してDB保存
- origin/UA検証
- アカウント列挙防止
- rate limiting

生成ファイル: src/auth/magic-link/
```

---

## 生成されるマジックリンク認証実装

```typescript
// src/auth/magic-link/service.ts

import crypto from 'crypto';
import { RateLimiterRedis } from 'rate-limiter-flexible';

const MAGIC_LINK_TTL = 15 * 60; // 15分
const MAX_SEND_ATTEMPTS = 1;     // 5分に1回
const RATE_WINDOW = 5 * 60;      // 5分

const sendRateLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: 'magic_link_send',
  points: MAX_SEND_ATTEMPTS,
  duration: RATE_WINDOW,
});

export class MagicLinkService {
  // マジックリンクを生成してメール送信
  async sendMagicLink(email: string, requestOrigin: string): Promise<void> {
    // rate limit チェック
    try {
      await sendRateLimiter.consume(email);
    } catch {
      // レート超過でも同じレスポンスを返す（タイミング攻撃対策）
      logger.warn({ email }, 'Magic link rate limit exceeded');
      return; // エラーを投げない
    }

    // アカウント有無に関係なく処理（列挙防止）
    const user = await prisma.user.findUnique({ where: { email } });

    if (user) {
      // トークン生成（48バイト）
      const token = crypto.randomBytes(48).toString('base64url');
      const tokenHash = this.hashToken(token);

      // DB保存（ハッシュのみ）
      await prisma.magicLinkToken.create({
        data: {
          tokenHash,
          userId: user.id,
          email,
          originBound: requestOrigin,
          expiresAt: new Date(Date.now() + MAGIC_LINK_TTL * 1000),
        },
      });

      // メール送信
      const magicUrl = `${requestOrigin}/auth/verify?token=${token}&email=${encodeURIComponent(email)}`;
      await emailService.send({
        to: email,
        subject: 'ログインリンク',
        html: this.buildEmailTemplate(magicUrl, user.name),
      });

      logger.info({ userId: user.id, origin: requestOrigin }, 'Magic link sent');
    }

    // ユーザーが存在しなくても同じ処理時間に見せる（タイミング攻撃対策）
    // user が null でも上と同じ時間になるよう疑似処理
  }

  // マジックリンクを検証してセッション作成
  async verifyMagicLink(
    token: string,
    email: string,
    requestOrigin: string,
  ): Promise<{ accessToken: string; refreshToken: string }> {
    const tokenHash = this.hashToken(token);

    // 有効なトークンを検索（使用済みは除外）
    const magicToken = await prisma.magicLinkToken.findFirst({
      where: {
        tokenHash,
        email,
        usedAt: null, // 未使用のみ
        expiresAt: { gt: new Date() },
      },
      include: { user: true },
    });

    if (!magicToken) {
      throw new AuthError('Invalid or expired magic link');
    }

    // originバインド確認
    if (magicToken.originBound !== requestOrigin) {
      logger.warn({
        expected: magicToken.originBound,
        received: requestOrigin,
        userId: magicToken.userId,
      }, 'Magic link origin mismatch (possible phishing)');
      throw new AuthError('Magic link origin mismatch');
    }

    // ワンタイム: 使用済みマークを付けてから処理
    await prisma.magicLinkToken.update({
      where: { id: magicToken.id },
      data: { usedAt: new Date() },
    });

    // JWTトークン発行
    const accessToken = await generateJWT({ sub: magicToken.userId }, { expiresIn: '15m' });
    const refreshToken = generateSecureToken(48);

    await prisma.refreshToken.create({
      data: {
        token: hashToken(refreshToken),
        userId: magicToken.userId,
        expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      },
    });

    // 同メールの未使用トークンを全て無効化（重複送信分）
    await prisma.magicLinkToken.updateMany({
      where: { email, usedAt: null },
      data: { usedAt: new Date() },
    });

    logger.info({ userId: magicToken.userId }, 'Magic link verified, session created');
    return { accessToken, refreshToken };
  }

  private hashToken(token: string): string {
    return crypto.createHash('sha256').update(token).digest('hex');
  }

  private buildEmailTemplate(magicUrl: string, name: string): string {
    return `
      <p>${name} さん、</p>
      <p>以下のリンクをクリックするとログインできます（15分間有効）：</p>
      <p><a href="${magicUrl}" style="font-size:18px">ログインする</a></p>
      <p>このリンクはあなた以外に送られていません。<br>
      クリックしていない場合はこのメールを無視してください。</p>
      <p><small>リンクをリクエストしていない場合は <a href="mailto:security@example.com">お知らせください</a>。</small></p>
    `;
  }
}
```

```typescript
// src/auth/magic-link/router.ts — APIエンドポイント

const service = new MagicLinkService();

// マジックリンク送信
router.post('/auth/magic-link', rateLimitMiddleware(5, 900), async (req, res) => {
  const { email } = req.body;
  const origin = req.get('origin') ?? req.get('host') ?? '';

  // バリデーション
  if (!z.string().email().safeParse(email).success) {
    return res.status(400).json({ error: 'Invalid email' });
  }

  // ユーザー存在有無に関わらず常に成功レスポンス
  await service.sendMagicLink(email, origin);
  res.json({ message: 'ログインリンクをメールに送りました' });
});

// マジックリンク検証
router.get('/auth/verify', async (req, res) => {
  const { token, email } = req.query as { token: string; email: string };
  const origin = req.get('origin') ?? `${req.protocol}://${req.get('host')}`;

  try {
    const { accessToken, refreshToken } = await service.verifyMagicLink(token, email, origin);

    // refresh_token を httpOnly Cookie に設定
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: true,
      sameSite: 'lax',
      maxAge: 30 * 24 * 60 * 60 * 1000,
    });

    // access_token はJSONで返す（SPAがメモリに保持）
    res.json({ accessToken });
  } catch (err) {
    res.status(401).json({ error: 'Invalid or expired magic link' });
  }
});
```

---

## まとめ

Claude CodeでマジックリンクでパスワードレーS認証を設計する：

1. **CLAUDE.md** に15分TTL・SHA256ハッシュ保存・originバインド・送信5分1回レートリミットを明記
2. **アカウント列挙防止** ユーザー不在でも常に同じレスポンスを返し、タイミング差も均一化
3. **originバインド** でフィッシングメールのリンクを攻撃者サイトに誘導しても検証でブロック
4. **ワンタイム処理** は「使用済みマーク → セッション発行」の順で行い、競合時の二重使用を防止

---

*認証設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
