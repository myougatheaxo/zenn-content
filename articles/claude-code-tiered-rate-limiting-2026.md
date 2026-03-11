---
title: "Claude Codeでプランティア別レート制限を設計する：Free/Pro/Enterprise・Redis Sliding Window"
emoji: "🎯"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "saas"]
published: true
published_at: "2026-03-16 11:00"
---

## はじめに

「Free/Pro/Enterpriseプランで呼び出し上限が違う」——プランティアに応じたレート制限をSlidingWindowアルゴリズムとRedisで実装し、超過時の明確なエラー（429 + Retry-After）を返す設計をClaude Codeに生成させる。

---

## CLAUDE.mdにティア別レート制限設計ルールを書く

```markdown
## プランティア別レート制限設計ルール

### レート制限値
| プラン | 1分 | 1時間 | 1日 |
|--------|-----|-------|-----|
| free   | 20  | 100   | 500 |
| pro    | 100 | 1,000 | 10,000 |
| enterprise | 1,000 | 50,000 | 制限なし |

### アルゴリズム
- Sliding Window（スライディングウィンドウ）: 固定ウィンドウより均一な制限が可能
- Redisのsorted setで実装（タイムスタンプをスコアとして使用）
- レスポンスヘッダー: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, Retry-After

### 超過時の挙動
- 429 Too Many Requests + Retry-Afterヘッダー（秒単位）
- エラーレスポンスにretry_afterフィールドを含める
- 超過ログをRedisに記録（監視・課金アップセル通知用）
```

---

## ティア別レート制限の生成

```
プランティア別レート制限システムを設計してください。

要件：
- Sliding Window（Redisのsorted set）
- Free/Pro/Enterpriseの制限値管理
- 429 + Retry-Afterヘッダー
- Prometheusメトリクス

生成ファイル: src/rateLimit/
```

---

## 生成されるティア別レート制限実装

```typescript
// src/rateLimit/tierLimits.ts — プランティア定義

export type PlanTier = 'free' | 'pro' | 'enterprise';

export interface RateLimitWindow {
  windowMs: number;   // ウィンドウサイズ（ms）
  maxRequests: number;
}

export const TIER_LIMITS: Record<PlanTier, RateLimitWindow[]> = {
  free: [
    { windowMs: 60_000,      maxRequests: 20    }, // 1分: 20req
    { windowMs: 3_600_000,   maxRequests: 100   }, // 1時間: 100req
    { windowMs: 86_400_000,  maxRequests: 500   }, // 1日: 500req
  ],
  pro: [
    { windowMs: 60_000,      maxRequests: 100   }, // 1分: 100req
    { windowMs: 3_600_000,   maxRequests: 1_000 }, // 1時間: 1,000req
    { windowMs: 86_400_000,  maxRequests: 10_000 }, // 1日: 10,000req
  ],
  enterprise: [
    { windowMs: 60_000,      maxRequests: 1_000  }, // 1分: 1,000req
    { windowMs: 3_600_000,   maxRequests: 50_000 }, // 1時間: 50,000req
    // 1日: 無制限
  ],
};
```

```typescript
// src/rateLimit/slidingWindowCounter.ts — Sliding Window実装

export interface RateLimitResult {
  allowed: boolean;
  limit: number;
  remaining: number;
  resetAt: Date;     // このウィンドウのリセット時刻
  retryAfterMs?: number;
}

export class SlidingWindowRateLimiter {
  async check(
    key: string,
    window: RateLimitWindow
  ): Promise<RateLimitResult> {
    const now = Date.now();
    const windowStart = now - window.windowMs;
    const redisKey = `rl:sw:${key}:${window.windowMs}`;

    // Lua スクリプトで原子的に実行（MULTI/EXECより安全）
    const luaScript = `
      local key = KEYS[1]
      local now = tonumber(ARGV[1])
      local window_start = tonumber(ARGV[2])
      local max_requests = tonumber(ARGV[3])
      local window_ms = tonumber(ARGV[4])

      -- 期限切れエントリを削除
      redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

      -- 現在のカウント
      local count = redis.call('ZCARD', key)

      if count < max_requests then
        -- 許可: 現在のタイムスタンプを追加（スコア=timestamp, メンバー=uuid）
        redis.call('ZADD', key, now, now .. '-' .. math.random(1000000))
        redis.call('PEXPIRE', key, window_ms)
        return {1, max_requests - count - 1} -- {allowed, remaining}
      else
        -- 拒否: 最古のエントリのタイムスタンプを返す
        local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
        return {0, tonumber(oldest[2])}
      end
    `;

    const result = await redis.eval(
      luaScript,
      1,
      [redisKey],
      [now.toString(), windowStart.toString(), window.maxRequests.toString(), window.windowMs.toString()]
    ) as [number, number];

    const [allowed, value] = result;

    if (allowed === 1) {
      return {
        allowed: true,
        limit: window.maxRequests,
        remaining: value,
        resetAt: new Date(now + window.windowMs),
      };
    } else {
      // valueは最古エントリのタイムスタンプ → リトライ可能時刻
      const oldestTs = value;
      const retryAfterMs = Math.max(0, oldestTs + window.windowMs - now);
      return {
        allowed: false,
        limit: window.maxRequests,
        remaining: 0,
        resetAt: new Date(oldestTs + window.windowMs),
        retryAfterMs,
      };
    }
  }

  // 複数ウィンドウを全てチェック（一つでも超過したら拒否）
  async checkAll(
    userId: string,
    tier: PlanTier,
    endpoint?: string // エンドポイントごとの追加制限
  ): Promise<{ allowed: boolean; mostRestrictive?: RateLimitResult }> {
    const limits = TIER_LIMITS[tier];
    if (!limits || limits.length === 0) return { allowed: true }; // enterprise無制限

    const results = await Promise.all(
      limits.map(window => this.check(`${userId}:${endpoint ?? 'global'}`, window))
    );

    // 一つでも拒否があれば全体を拒否
    const denied = results.find(r => !r.allowed);
    if (denied) {
      metrics.counter('rate_limit_exceeded', 1, { tier, endpoint: endpoint ?? 'global' });
      return { allowed: false, mostRestrictive: denied };
    }

    // 最も制限が厳しいウィンドウのRemainingを返す
    const mostRestrictive = results.reduce((a, b) => a.remaining < b.remaining ? a : b);
    return { allowed: true, mostRestrictive };
  }
}
```

```typescript
// src/rateLimit/middleware.ts — Expressミドルウェア

const rateLimiter = new SlidingWindowRateLimiter();

export function rateLimitByTier(endpoint?: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    // 認証不要エンドポイントはIPベースでfreeティア制限
    const userId = req.userId ?? `ip:${req.ip}`;
    const tier: PlanTier = req.user?.subscription?.plan ?? 'free';

    const { allowed, mostRestrictive } = await rateLimiter.checkAll(userId, tier, endpoint);

    // レスポンスヘッダーを常に付与
    if (mostRestrictive) {
      res.set({
        'X-RateLimit-Limit': mostRestrictive.limit.toString(),
        'X-RateLimit-Remaining': mostRestrictive.remaining.toString(),
        'X-RateLimit-Reset': Math.floor(mostRestrictive.resetAt.getTime() / 1000).toString(),
      });
    }

    if (!allowed && mostRestrictive) {
      const retryAfterSec = Math.ceil((mostRestrictive.retryAfterMs ?? 60_000) / 1000);
      res.set('Retry-After', retryAfterSec.toString());

      logger.info({ userId, tier, endpoint, retryAfterSec }, 'Rate limit exceeded');

      return res.status(429).json({
        error: 'Too Many Requests',
        message: `Rate limit exceeded for ${tier} plan. Please upgrade or wait.`,
        retry_after: retryAfterSec,
        upgrade_url: tier !== 'enterprise' ? 'https://example.com/pricing' : undefined,
      });
    }

    next();
  };
}

// 使用例
router.get('/api/data', requireAuth, rateLimitByTier('data-export'), async (req, res) => {
  const data = await dataService.export(req.userId);
  res.json(data);
});

// プランアップセルトリガー（超過頻度に応じて通知）
async function checkUpselOpportunity(userId: string, tier: PlanTier): Promise<void> {
  if (tier === 'enterprise') return;

  const exceeded = await redis.incr(`rl:exceeded:${userId}`);
  await redis.expire(`rl:exceeded:${userId}`, 86400);

  if (exceeded === 3) { // 1日3回超過でアップセル通知
    await notificationService.sendUpgradePrompt(userId, tier);
  }
}
```

---

## まとめ

Claude Codeでプランティア別レート制限を設計する：

1. **CLAUDE.md** にFree/Pro/Enterpriseの1分・1時間・1日制限値の数値表・429+Retry-Afterヘッダー必須・超過ログ記録を明記
2. **Sliding Window（Sorted Set）** でRedis Luaスクリプトを原子実行——固定ウィンドウのバースト問題を解消し均一な制限を実現
3. **複数ウィンドウ全チェック** で短期バースト（1分）と長期消費（1日）を同時制限——Pro以上でも1分100reqを超えたら429
4. **超過頻度でアップセルトリガー** ——1日3回超過でアップグレード通知メール→収益につながる機会として活用

---

*SaaS設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
