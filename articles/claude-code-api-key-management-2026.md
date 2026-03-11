---
title: "Claude CodeでAPIキー管理を設計する：スコープ・ローテーション・使用量追跡"
emoji: "🗝️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "security", "redis"]
published: true
published_at: "2026-03-15 12:00"
---

## はじめに

「本番APIキーを開発者が全員共有」「流出したキーをすぐに無効化できない」——スコープ付きAPIキー・即時失効・使用量追跡をClaude Codeに設計させる。

---

## CLAUDE.mdにAPIキー管理設計ルールを書く

```markdown
## APIキー管理設計ルール

### キー設計
- フォーマット: {prefix}_{random64bytes_base62}（例: sk_live_xxxxx）
- DBにはSHA256ハッシュのみ保存（プレーンは初回のみ表示）
- prefix でキーの種別を識別: sk（secret）, pk（public）, rk（restricted）

### スコープ管理
- スコープリスト: orders:read, orders:write, users:read, admin:*
- キーごとにスコープを制限（最小権限）
- スコープは作成時に固定（変更不可、再作成が必要）

### 使用量・監視
- 使用ごとにRedisカウンター更新（minute/hour/dayバケット）
- 不審なアクセスパターン: 1分100回超 → アラート
- キーのIPバインド（オプション）: 特定IPのみ使用可
```

---

## APIキー管理の生成

```
スコープ付きAPIキー管理システムを設計してください。

要件：
- スコープ付きキー生成
- ハッシュ保存
- 即時失効
- 使用量追跡
- Redis認証キャッシュ

生成ファイル: src/auth/apiKey/
```

---

## 生成されるAPIキー管理実装

```typescript
// src/auth/apiKey/generator.ts — キー生成

import crypto from 'crypto';

type KeyPrefix = 'sk_live' | 'sk_test' | 'rk'; // live/test/restricted

interface GeneratedKey {
  plaintext: string;  // 一度だけユーザーに表示（保存しない）
  prefix: string;
  hash: string;       // DBに保存
  hint: string;       // 末尾4文字（UI表示用）
}

export function generateAPIKey(prefix: KeyPrefix): GeneratedKey {
  // 64バイトのランダム値をBase62エンコード
  const random = crypto.randomBytes(48).toString('base64url');
  const plaintext = `${prefix}_${random}`;

  const hash = crypto.createHash('sha256').update(plaintext).digest('hex');
  const hint = plaintext.slice(-4);

  return { plaintext, prefix, hash, hint };
}

// src/auth/apiKey/service.ts — キー管理サービス

export class APIKeyService {
  async createKey(options: {
    userId: string;
    name: string;
    scopes: string[];
    expiresAt?: Date;
    allowedIPs?: string[];
  }): Promise<{ key: APIKey; plaintext: string }> {
    // スコープのバリデーション
    const validScopes = await this.getAvailableScopes(options.userId);
    const invalidScopes = options.scopes.filter(s => !this.isScopeGranted(s, validScopes));
    if (invalidScopes.length > 0) {
      throw new ValidationError(`Invalid scopes: ${invalidScopes.join(', ')}`);
    }

    const generated = generateAPIKey('sk_live');

    const key = await prisma.apiKey.create({
      data: {
        userId: options.userId,
        name: options.name,
        keyHash: generated.hash,
        keyHint: generated.hint,
        scopes: options.scopes,
        expiresAt: options.expiresAt,
        allowedIPs: options.allowedIPs ?? [],
        lastUsedAt: null,
        revokedAt: null,
      },
    });

    logger.info({ userId: options.userId, keyId: key.id, scopes: options.scopes }, 'API key created');

    // 作成時のみplaインテキストを返す（以降は取得不可）
    return { key, plaintext: generated.plaintext };
  }

  // キーの認証（認証ミドルウェアから呼ばれる）
  async authenticateKey(plaintext: string, requiredScope: string): Promise<{
    valid: boolean;
    userId?: string;
    keyId?: string;
  }> {
    const hash = crypto.createHash('sha256').update(plaintext).digest('hex');

    // Redisキャッシュを先に確認（1分TTL）
    const cacheKey = `apikey:${hash}`;
    const cached = await redis.get(cacheKey);

    let keyData: APIKeyCache | null = cached ? JSON.parse(cached) : null;

    if (!keyData) {
      const key = await prisma.apiKey.findFirst({
        where: {
          keyHash: hash,
          revokedAt: null,
          OR: [
            { expiresAt: null },
            { expiresAt: { gt: new Date() } },
          ],
        },
        select: { id: true, userId: true, scopes: true, allowedIPs: true },
      });

      if (!key) return { valid: false };

      keyData = { id: key.id, userId: key.userId, scopes: key.scopes, allowedIPs: key.allowedIPs };
      await redis.set(cacheKey, JSON.stringify(keyData), { EX: 60 });
    }

    // スコープチェック
    if (!this.isScopeGranted(requiredScope, keyData.scopes)) {
      logger.warn({ keyId: keyData.id, required: requiredScope, granted: keyData.scopes }, 'Scope denied');
      return { valid: false };
    }

    // 使用量記録（非同期）
    setImmediate(() => this.trackUsage(keyData!.id, keyData!.userId));

    return { valid: true, userId: keyData.userId, keyId: keyData.id };
  }

  // スコープマッチング（ワイルドカード対応）
  private isScopeGranted(required: string, granted: string[]): boolean {
    return granted.some(g => {
      if (g === 'admin:*') return true; // 全権限
      if (g === required) return true;  // 完全一致
      if (g.endsWith(':*')) {
        const namespace = g.slice(0, -2);
        return required.startsWith(`${namespace}:`);
      }
      return false;
    });
  }

  // 即時失効（Redisキャッシュも削除）
  async revokeKey(keyId: string, userId: string): Promise<void> {
    const key = await prisma.apiKey.findFirst({ where: { id: keyId, userId } });
    if (!key) throw new NotFoundError('API key not found');

    await prisma.apiKey.update({
      where: { id: keyId },
      data: { revokedAt: new Date() },
    });

    // Redisキャッシュを即座に無効化
    await redis.del(`apikey:${key.keyHash}`);
    logger.info({ keyId, userId }, 'API key revoked');
  }

  private async trackUsage(keyId: string, userId: string): Promise<void> {
    const now = Date.now();
    const minBucket = Math.floor(now / 60_000);
    const hourBucket = Math.floor(now / 3_600_000);
    const dayBucket = Math.floor(now / 86_400_000);

    const pipeline = redis.pipeline();
    pipeline.incr(`apikey:usage:min:${keyId}:${minBucket}`);
    pipeline.expire(`apikey:usage:min:${keyId}:${minBucket}`, 120);
    pipeline.incr(`apikey:usage:hour:${keyId}:${hourBucket}`);
    pipeline.expire(`apikey:usage:hour:${keyId}:${hourBucket}`, 7200);
    pipeline.incr(`apikey:usage:day:${keyId}:${dayBucket}`);
    pipeline.expire(`apikey:usage:day:${keyId}:${dayBucket}`, 172800);
    await pipeline.exec();

    // 1分100回超はアラート
    const minuteCount = parseInt(await redis.get(`apikey:usage:min:${keyId}:${minBucket}`) ?? '0');
    if (minuteCount > 100) {
      Sentry.captureMessage('API key rate anomaly detected', {
        level: 'warning',
        extra: { keyId, userId, minuteCount },
      });
    }

    // last_used_at 更新（1分に1回のみ）
    await redis.set(`apikey:lastused:${keyId}`, String(now), { EX: 70 });
    if (!(await redis.get(`apikey:lastused_db:${keyId}`))) {
      await prisma.apiKey.update({ where: { id: keyId }, data: { lastUsedAt: new Date() } });
      await redis.set(`apikey:lastused_db:${keyId}`, '1', { EX: 60 });
    }
  }
}
```

```typescript
// src/auth/apiKey/middleware.ts — Express統合

export function requireAPIKey(scope: string) {
  const service = new APIKeyService();

  return async (req: Request, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;
    const key = authHeader?.replace('Bearer ', '') ?? req.headers['x-api-key'] as string;

    if (!key) return res.status(401).json({ error: 'API key required' });

    const result = await service.authenticateKey(key, scope);
    if (!result.valid) return res.status(403).json({ error: 'Invalid key or insufficient scope' });

    req.userId = result.userId!;
    req.apiKeyId = result.keyId!;
    next();
  };
}

// 使用例
router.get('/api/orders', requireAPIKey('orders:read'), async (req, res) => {
  const orders = await prisma.order.findMany({ where: { userId: req.userId } });
  res.json(orders);
});
```

---

## まとめ

Claude CodeでAPIキー管理を設計する：

1. **CLAUDE.md** にSHA256ハッシュ保存・初回のみplaintext表示・スコープ固定・Redisキャッシュ1分を明記
2. **prefix_random64** フォーマットでキー種別を視認可能——`sk_live_` なら本番キーとすぐわかる
3. **Redis 1分キャッシュ** で毎リクエストのDB参照を回避——revoke時はキャッシュも即削除
4. **使用量追跡** はmin/hour/dayバケットでRedisに記録——1分100回超で自動アラート、last_used_atは1分に1回のDB更新

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
