---
title: "Claude CodeでCache-Asideパターンを設計する：キャッシュ戦略・整合性・TTL管理"
emoji: "🗄️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "postgresql"]
published: true
published_at: "2026-03-17 14:00"
---

## はじめに

「キャッシュの中身が古くなって不整合が起きた」「TTLを長くしたら古いデータを返し続ける」——Cache-AsideパターンでDB参照コストを下げながら整合性を保つ設計をClaude Codeに生成させる。

---

## CLAUDE.mdにCache-Aside設計ルールを書く

```markdown
## Cache-Aside設計ルール

### キャッシュ読み取り
1. Redisを参照 → ヒットなら返す
2. ミスならDBを参照 → Redisに書き込んで返す
3. DBミスもRedisにキャッシュ（Negative Cache）してリクエストの波を防ぐ

### キャッシュ無効化
- 書き込み時にキャッシュを削除（Write-Invalidate）
- キャッシュ更新は書かず削除のみ（更新競合を回避）
- TTLは最悪ケースの不整合許容時間で決める（例: ユーザープロフィール = 5分）

### ホットスポット対策
- 人気キーは確率的TTL（95-100%の間でランダム化）で同時期限切れを防止
- キャッシュウォームアップ: デプロイ直後にDBからプリロード
- ローカルキャッシュ（Node.jsメモリ）+ Redisの2層構成でレイテンシー削減
```

---

## Cache-Aside実装の生成

```
Cache-Asideパターンを設計してください。

要件：
- キャッシュミス時のDB参照と自動キャッシュ
- 書き込み時の整合性維持
- ネガティブキャッシュ
- 2層キャッシュ（メモリ + Redis）

生成ファイル: src/cache/cacheAside/
```

---

## 生成されるCache-Aside実装

```typescript
// src/cache/cacheAside/cacheAsideRepository.ts — Cache-Asideリポジトリ

export interface CacheConfig {
  keyPrefix: string;
  ttlSec: number;
  negativeCacheTtlSec?: number;  // Negative Cache TTL（デフォルト: 30秒）
  localCacheTtlMs?: number;      // ローカルキャッシュTTL（デフォルト: 5秒）
  jitterPercent?: number;         // TTLジッター（0-100%）でスタンピード防止
}

const NEGATIVE_CACHE_SENTINEL = '__NULL__'; // DBミスを区別するマーカー

export class CacheAsideRepository<T> {
  private readonly localCache = new Map<string, { value: T | null; expiresAt: number }>();
  private localCacheHits = 0;
  private redisCacheHits = 0;
  private cacheMisses = 0;

  constructor(private readonly config: CacheConfig) {}

  async get(key: string, fetcher: () => Promise<T | null>): Promise<T | null> {
    const fullKey = `${this.config.keyPrefix}:${key}`;

    // 1. ローカルキャッシュ（Node.jsメモリ）
    const localCached = this.getFromLocalCache(fullKey);
    if (localCached !== undefined) {
      this.localCacheHits++;
      return localCached;
    }

    // 2. Redis
    const redisCached = await redis.get(fullKey);
    if (redisCached !== null) {
      this.redisCacheHits++;

      if (redisCached === NEGATIVE_CACHE_SENTINEL) {
        this.setLocalCache(fullKey, null);
        return null; // Negative cache hit
      }

      const value = JSON.parse(redisCached) as T;
      this.setLocalCache(fullKey, value);
      return value;
    }

    // 3. DB（キャッシュミス）
    this.cacheMisses++;
    const value = await fetcher();

    if (value === null) {
      // Negative Cache（DBにも存在しない場合）
      const negativeTtl = this.config.negativeCacheTtlSec ?? 30;
      await redis.set(fullKey, NEGATIVE_CACHE_SENTINEL, { EX: negativeTtl });
      this.setLocalCache(fullKey, null);
    } else {
      // TTLにジッターを追加（スタンピード防止）
      const ttl = this.applyJitter(this.config.ttlSec);
      await redis.set(fullKey, JSON.stringify(value), { EX: ttl });
      this.setLocalCache(fullKey, value);
    }

    return value;
  }

  // 書き込み時: キャッシュを削除（Write-Invalidate）
  async invalidate(key: string): Promise<void> {
    const fullKey = `${this.config.keyPrefix}:${key}`;
    await redis.del(fullKey);
    this.localCache.delete(fullKey);
    logger.debug({ key: fullKey }, 'Cache invalidated');
  }

  // 複数キーの一括無効化
  async invalidateMany(keys: string[]): Promise<void> {
    if (keys.length === 0) return;

    const fullKeys = keys.map(k => `${this.config.keyPrefix}:${k}`);
    await redis.del(fullKeys);
    fullKeys.forEach(k => this.localCache.delete(k));
  }

  // キャッシュメトリクス
  getStats(): { localHits: number; redisHits: number; misses: number; hitRate: number } {
    const total = this.localCacheHits + this.redisCacheHits + this.cacheMisses;
    return {
      localHits: this.localCacheHits,
      redisHits: this.redisCacheHits,
      misses: this.cacheMisses,
      hitRate: total > 0 ? (this.localCacheHits + this.redisCacheHits) / total : 0,
    };
  }

  private getFromLocalCache(key: string): T | null | undefined {
    const entry = this.localCache.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expiresAt) {
      this.localCache.delete(key);
      return undefined;
    }
    return entry.value;
  }

  private setLocalCache(key: string, value: T | null): void {
    const ttlMs = this.config.localCacheTtlMs ?? 5_000;
    this.localCache.set(key, { value, expiresAt: Date.now() + ttlMs });
  }

  private applyJitter(ttlSec: number): number {
    const jitterPercent = this.config.jitterPercent ?? 10;
    const jitter = (Math.random() * jitterPercent) / 100;
    return Math.floor(ttlSec * (1 + jitter));
  }
}
```

```typescript
// src/cache/cacheAside/userRepository.ts — ユーザーリポジトリ実装例

export class UserRepository {
  private readonly cache = new CacheAsideRepository<User>({
    keyPrefix: 'user',
    ttlSec: 300,          // 5分
    negativeCacheTtlSec: 30,
    localCacheTtlMs: 5_000,
    jitterPercent: 10,    // 270-300秒のランダムTTL
  });

  async findById(id: string): Promise<User | null> {
    return this.cache.get(id, () =>
      prisma.user.findUnique({ where: { id }, include: { profile: true } })
    );
  }

  async update(id: string, data: UpdateUserInput): Promise<User> {
    const updated = await prisma.user.update({ where: { id }, data });

    // Write-Invalidate: 書き込み後にキャッシュを削除（更新の書き込みは行わない）
    await this.cache.invalidate(id);

    return updated;
  }

  async delete(id: string): Promise<void> {
    await prisma.user.delete({ where: { id } });
    await this.cache.invalidate(id);
  }
}

// キャッシュウォームアップ（デプロイ直後に実行）
export async function warmUpUserCache(limit = 500): Promise<void> {
  logger.info('Starting cache warm-up for hot users');

  // 直近7日間にアクセスされたユーザーをプリロード
  const hotUsers = await prisma.user.findMany({
    where: { lastActiveAt: { gte: new Date(Date.now() - 7 * 86400_000) } },
    orderBy: { lastActiveAt: 'desc' },
    take: limit,
    include: { profile: true },
  });

  const pipeline = redis.pipeline();
  for (const user of hotUsers) {
    const key = `user:${user.id}`;
    pipeline.set(key, JSON.stringify(user), { EX: 300 });
  }

  await pipeline.exec();
  logger.info({ count: hotUsers.length }, 'Cache warm-up completed');
}
```

---

## まとめ

Claude CodeでCache-Asideパターンを設計する：

1. **CLAUDE.md** にキャッシュミス時のDB参照→Redis書き込み・書き込み時はキャッシュ削除（更新しない）・Negative CacheでDB不存在も30秒キャッシュを明記
2. **Write-Invalidate（更新ではなく削除）** で書き込みとキャッシュ更新の競合状態を回避——「DBに書いた後キャッシュに書く間に別リクエストが古い値を読む」問題を防止
3. **2層キャッシュ（Node.jsメモリ + Redis）** でレイテンシーを最小化——ローカルキャッシュのTTLは5秒（短い）、Redisは5分——ローカルの古さよりも速度を優先
4. **確率的TTLジッター（±10%）** でホットキーの同時期限切れを防止——1万件が同時刻に期限切れするとDBへのスパイクが発生するため、TTLをランダム化して分散

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
