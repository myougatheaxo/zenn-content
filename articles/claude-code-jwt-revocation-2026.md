---
title: "Claude CodeでJWTトークン無効化を設計する：ブラックリスト・トークンローテーション・セキュリティイベント連動"
emoji: "🚫"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-21 12:00"
---

## はじめに

「ユーザーをBANしてもJWTの有効期限が切れるまでアクセスできる」「パスワード変更後に旧トークンでアクセスされる」——JWTのステートレス特性を維持しながらトークンを即時無効化する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにJWT無効化設計ルールを書く

```markdown
## JWT無効化設計ルール

### なぜ難しいか
- JWTはサーバー側に状態を持たない（ステートレス）
- 一度発行したトークンは有効期限が来るまで有効
- ブラックリストを持つとステートレスの利点が失われる

### 戦略の選択
- Short-lived Token: アクセストークンを15分以内に（基本戦略）
- ブラックリスト: 特定トークンのJTIをRedisに記録（緊急無効化）
- Generation Counter: ユーザーのトークン世代をインクリメント（一括無効化）
- ローテーション: リフレッシュトークンを使い回し不可に（盗難検知）

### セキュリティイベント連動
- パスワード変更: そのユーザーの全トークンを無効化
- 不正ログイン検知: アカウントロック + 全トークン無効化
- デバイス削除: デバイス固有トークンを無効化
```

---

## JWT無効化実装の生成

```
JWT無効化システムを設計してください。

要件：
- トークン世代カウンター（一括無効化）
- JTIブラックリスト（個別無効化）
- セキュリティイベント連動
- デバイス別トークン管理

生成ファイル: src/auth/
```

---

## 生成されるJWT無効化実装

```typescript
// src/auth/tokenInvalidationService.ts — トークン無効化サービス

export interface TokenPayload {
  sub: string;       // ユーザーID
  jti: string;       // JWT ID（トークンを個別識別）
  gen: number;       // トークン世代（Generation Counter）
  deviceId?: string; // デバイス識別子
  iat: number;
  exp: number;
}

export class TokenInvalidationService {
  private readonly GENERATION_KEY = (userId: string) => `token:gen:${userId}`;
  private readonly BLACKLIST_KEY = (jti: string) => `token:blacklist:${jti}`;
  private readonly DEVICE_KEY = (userId: string, deviceId: string) =>
    `token:device:${userId}:${deviceId}`;

  constructor(
    private readonly redis: Redis,
    private readonly jwtService: JwtService
  ) {}

  // ===== トークン世代カウンター（一括無効化） =====

  // ユーザーの現在のトークン世代を取得
  async getTokenGeneration(userId: string): Promise<number> {
    const gen = await this.redis.get(this.GENERATION_KEY(userId));
    return gen ? parseInt(gen) : 0;
  }

  // 全トークンを一括無効化（パスワード変更・アカウントBAN時）
  async invalidateAllTokens(userId: string, reason: string): Promise<void> {
    const newGen = await this.redis.incr(this.GENERATION_KEY(userId));
    // TTL: リフレッシュトークンの最大有効期間（30日）
    await this.redis.expire(this.GENERATION_KEY(userId), 30 * 24 * 60 * 60);

    logger.security({
      userId,
      action: 'invalidate_all_tokens',
      newGeneration: newGen,
      reason,
    });

    // セキュリティイベント発行
    await this.eventPublisher.publish({
      type: 'security:tokens-invalidated',
      userId,
      reason,
      timestamp: new Date(),
    });
  }

  // ===== JTIブラックリスト（個別無効化） =====

  // 特定のトークンを無効化（ログアウト、デバイス削除）
  async blacklistToken(jti: string, expiresAt: Date): Promise<void> {
    const ttl = Math.ceil((expiresAt.getTime() - Date.now()) / 1000);
    if (ttl > 0) {
      // トークンの有効期限までブラックリストに追加
      await this.redis.setEx(this.BLACKLIST_KEY(jti), ttl, '1');
    }
  }

  async isBlacklisted(jti: string): Promise<boolean> {
    return (await this.redis.exists(this.BLACKLIST_KEY(jti))) === 1;
  }

  // ===== デバイス別トークン管理 =====

  async registerDevice(userId: string, deviceId: string, refreshToken: string): Promise<void> {
    const hashedToken = await bcrypt.hash(refreshToken, 10);
    await this.redis.setEx(
      this.DEVICE_KEY(userId, deviceId),
      30 * 24 * 60 * 60,  // 30日
      JSON.stringify({ hashedToken, registeredAt: new Date() })
    );
  }

  async revokeDevice(userId: string, deviceId: string): Promise<void> {
    await this.redis.del(this.DEVICE_KEY(userId, deviceId));
    logger.security({ userId, deviceId, action: 'revoke_device' });
  }

  async isDeviceRegistered(userId: string, deviceId: string, refreshToken: string): Promise<boolean> {
    const stored = await this.redis.get(this.DEVICE_KEY(userId, deviceId));
    if (!stored) return false;
    const { hashedToken } = JSON.parse(stored);
    return bcrypt.compare(refreshToken, hashedToken);
  }

  // ===== トークン検証（全チェックを統合） =====

  async validateToken(token: string): Promise<TokenPayload> {
    // 1. JWT署名検証と期限チェック
    const payload = this.jwtService.verify<TokenPayload>(token);

    // 2. JTIブラックリストチェック
    if (await this.isBlacklisted(payload.jti)) {
      throw new TokenBlacklistedError(payload.jti);
    }

    // 3. 世代チェック（一括無効化の確認）
    const currentGen = await this.getTokenGeneration(payload.sub);
    if (payload.gen < currentGen) {
      throw new TokenGenerationExpiredError(payload.sub, payload.gen, currentGen);
    }

    return payload;
  }
}
```

```typescript
// src/auth/securityEventHandler.ts — セキュリティイベント連動

export class SecurityEventHandler {
  constructor(
    private readonly tokenService: TokenInvalidationService,
    private readonly sessionService: SessionService
  ) {}

  // パスワード変更: 全デバイスのトークンを無効化
  async onPasswordChanged(userId: string): Promise<void> {
    await Promise.all([
      this.tokenService.invalidateAllTokens(userId, 'password_changed'),
      this.sessionService.terminateAllSessions(userId),
    ]);

    logger.security({ userId, event: 'password_changed', action: 'all_tokens_invalidated' });
  }

  // 不正ログイン検知: アカウントロック + 全トークン無効化
  async onSuspiciousActivity(userId: string, ipAddress: string): Promise<void> {
    await Promise.all([
      this.tokenService.invalidateAllTokens(userId, 'suspicious_activity'),
      this.accountService.temporaryLock(userId, 24 * 60 * 60),  // 24時間ロック
      this.notificationService.sendSecurityAlert(userId, {
        event: 'suspicious_activity_detected',
        ipAddress,
        timestamp: new Date(),
      }),
    ]);
  }

  // ログアウト（デバイス指定）
  async onLogout(userId: string, jti: string, deviceId: string, expiresAt: Date): Promise<void> {
    await Promise.all([
      this.tokenService.blacklistToken(jti, expiresAt),
      this.tokenService.revokeDevice(userId, deviceId),
    ]);
  }

  // リフレッシュトークンの使い回し検知（Token Rotation）
  async onRefreshTokenReuse(userId: string, deviceId: string): Promise<void> {
    // リフレッシュトークンの使い回しはトークン盗難の可能性
    logger.security({
      userId,
      deviceId,
      event: 'refresh_token_reuse_detected',
      severity: 'critical',
    });

    // 全デバイスのトークンを無効化（攻撃者も被害者も両方ログアウト）
    await this.tokenService.invalidateAllTokens(userId, 'refresh_token_reuse_detected');
    await this.notificationService.sendSecurityAlert(userId, {
      event: 'token_theft_suspected',
      action: 'all_sessions_terminated',
    });
  }
}

// 認証ミドルウェア統合
export async function authMiddleware(req: Request, res: Response, next: NextFunction): Promise<void> {
  const token = extractBearerToken(req);
  if (!token) {
    return res.status(401).json({ code: 'UNAUTHORIZED' });
  }

  try {
    const payload = await tokenInvalidationService.validateToken(token);
    req.user = { id: payload.sub, deviceId: payload.deviceId };
    next();
  } catch (error) {
    if (error instanceof TokenBlacklistedError || error instanceof TokenGenerationExpiredError) {
      return res.status(401).json({ code: 'TOKEN_REVOKED', message: 'Token has been revoked' });
    }
    return res.status(401).json({ code: 'INVALID_TOKEN' });
  }
}
```

---

## まとめ

Claude CodeでJWTトークン無効化を設計する：

1. **CLAUDE.md** にアクセストークン15分以内・世代カウンターで一括無効化・JTIブラックリストで個別無効化・リフレッシュトークン使い回し検知で全セッション終了を明記
2. **Generation Counter（世代カウンター）** でパスワード変更後の全トークンを即座に無効化——Redisに`token:gen:{userId}`を保存し、パスワード変更時にインクリメント。トークンの`gen`フィールドが現在の世代より低ければ無効
3. **JTIブラックリスト** でトークンの有効期限まで個別無効化——`jti`（JWT ID）はulid()で生成し、ログアウト時にRedisに追加。TTLをトークンの有効期限に合わせることでRedisが自動クリーンアップ
4. **リフレッシュトークンローテーションで盗難検知** ——使い回された（reuse）リフレッシュトークンを検知したら「攻撃者が使ったか被害者が使ったか分からない」として全デバイスのトークンを一括無効化しセキュリティアラートを送信

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
