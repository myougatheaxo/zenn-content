---
title: "Claude Codeでキャッシュスタンピード対策を設計する：確率的早期期限・Mutex Lock・バックグラウンド更新"
emoji: "🏇"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "performance"]
published: true
published_at: "2026-03-14 10:00"
---

## はじめに

人気記事のキャッシュが切れた瞬間に100リクエストが同時にDBへ——キャッシュスタンピード（Thundering Herd問題）の3種の対策をClaude Codeに設計させる。

---

## CLAUDE.mdにキャッシュスタンピード設計ルールを書く

```markdown
## キャッシュスタンピード設計ルール

### 対策3選（用途別）
1. 確率的早期更新（PER）: TTLが近づくとランダムに更新開始→ 一般的なキャッシュに最適
2. Mutex Lock: 最初の1リクエストのみDB参照、他は待機→ 計算コスト高いクエリ
3. バックグラウンド非同期更新: 古いキャッシュを即座に返しつつ更新開始→ レイテンシ最優先

### キャッシュTTL設計
- 固定TTL + ランダムジッター（-30%〜+30%）で同時期限切れ防止
- 重要データ: stale-while-revalidate パターン
- Redisクラスター: TTL分散のためハッシュタグを避ける

### 計測
- cache_hit_ratio > 0.9 を目標
- キャッシュミス時のレイテンシスパイクをPrometheusで記録
```

---

## キャッシュスタンピード対策の生成

```
キャッシュスタンピード（Thundering Herd）対策を設計してください。

要件：
- 確率的早期更新（PER）
- Mutex Lockパターン
- Stale-While-Revalidate
- TTLジッター

生成ファイル: src/cache/
```

---

## 生成されるキャッシュスタンピード対策実装

```typescript
// src/cache/stampedePrevention.ts

// ===================================
// 1. 確率的早期更新（PER: Probabilistic Early Recomputation）
// ===================================
// TTLが近づくにつれて、確率的に早期更新を開始する
// beta=1.0: 標準、beta>1: より積極的に早期更新

export async function getWithPER<T>(
  key: string,
  fetchFn: () => Promise<T>,
  options: {
    ttlSeconds: number;
    beta?: number; // 1.0 がデフォルト
  }
): Promise<T> {
  const beta = options.beta ?? 1.0;

  // キャッシュとTTLを同時取得
  const [cached, ttlRemaining] = await Promise.all([
    redis.get(key),
    redis.ttl(key),
  ]);

  if (cached !== null && ttlRemaining > 0) {
    // 確率的早期更新の判定
    // 残TTLが少ないほど更新確率が高くなる
    const earlyRefreshProb = Math.exp(-ttlRemaining / (beta * options.ttlSeconds));
    const shouldEarlyRefresh = Math.random() < earlyRefreshProb;

    if (!shouldEarlyRefresh) {
      return JSON.parse(cached);
    }
    // 早期更新決定 → フォールスルーして再計算
    logger.debug({ key, ttlRemaining }, 'PER: early refresh triggered');
  }

  // 計算・取得してキャッシュに保存
  const data = await fetchFn();
  const jitteredTtl = Math.round(options.ttlSeconds * (0.8 + Math.random() * 0.4)); // ±20%ジッター
  await redis.set(key, JSON.stringify(data), { EX: jitteredTtl });
  return data;
}

// ===================================
// 2. Mutex Lock（1リクエストのみDB参照）
// ===================================
export async function getWithMutex<T>(
  key: string,
  fetchFn: () => Promise<T>,
  options: {
    ttlSeconds: number;
    lockTimeoutMs?: number;
    waitIntervalMs?: number;
  }
): Promise<T> {
  const lockKey = `lock:${key}`;
  const lockTimeoutMs = options.lockTimeoutMs ?? 10_000;
  const waitIntervalMs = options.waitIntervalMs ?? 50;

  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached);

  // Redisで分散ロック取得
  const lockAcquired = await redis.set(lockKey, '1', {
    NX: true, // 存在しない場合のみ設定
    PX: lockTimeoutMs, // ロックの最大保持時間
  });

  if (lockAcquired) {
    // ロック取得成功: このリクエストがDB参照を担当
    try {
      const data = await fetchFn();
      const jitteredTtl = Math.round(options.ttlSeconds * (0.9 + Math.random() * 0.2));
      await redis.set(key, JSON.stringify(data), { EX: jitteredTtl });
      return data;
    } finally {
      await redis.del(lockKey); // 必ずロック解放
    }
  } else {
    // ロック取得失敗: 他リクエストがキャッシュを作成するまで待機
    const deadline = Date.now() + lockTimeoutMs;

    while (Date.now() < deadline) {
      await sleep(waitIntervalMs);
      const value = await redis.get(key);
      if (value !== null) return JSON.parse(value);
    }

    // タイムアウト: 直接DBを参照（フォールバック）
    logger.warn({ key }, 'Mutex lock timeout, falling back to direct fetch');
    return fetchFn();
  }
}

// ===================================
// 3. Stale-While-Revalidate（古いデータを即返す）
// ===================================
interface CacheEntry<T> {
  data: T;
  cachedAt: number;
  ttlMs: number;
}

export async function getWithSWR<T>(
  key: string,
  fetchFn: () => Promise<T>,
  options: {
    freshTtlMs: number;   // この時間内は完全にフレッシュ（再計算なし）
    staleTtlMs: number;   // この時間内は古いデータを返しながらバックグラウンド更新
  }
): Promise<T> {
  const rawEntry = await redis.get(key);

  if (rawEntry !== null) {
    const entry: CacheEntry<T> = JSON.parse(rawEntry);
    const ageMs = Date.now() - entry.cachedAt;

    if (ageMs < options.freshTtlMs) {
      // フレッシュ: そのまま返す
      return entry.data;
    }

    if (ageMs < options.staleTtlMs) {
      // Stale: 古いデータを即座に返しつつ、バックグラウンドで更新
      setImmediate(async () => {
        try {
          const newData = await fetchFn();
          const newEntry: CacheEntry<T> = {
            data: newData,
            cachedAt: Date.now(),
            ttlMs: options.staleTtlMs,
          };
          await redis.set(key, JSON.stringify(newEntry), { PX: options.staleTtlMs });
        } catch (err) {
          logger.error({ key, err }, 'Background revalidation failed');
        }
      });
      return entry.data; // 古いデータを即返す
    }
  }

  // キャッシュなし or 期限切れ: 同期で取得
  const data = await fetchFn();
  const entry: CacheEntry<T> = { data, cachedAt: Date.now(), ttlMs: options.staleTtlMs };
  await redis.set(key, JSON.stringify(entry), { PX: options.staleTtlMs });
  return data;
}

// 使用例
export async function getPopularArticles(): Promise<Article[]> {
  return getWithSWR(
    'articles:popular',
    () => prisma.article.findMany({ orderBy: { viewCount: 'desc' }, take: 10 }),
    { freshTtlMs: 60_000, staleTtlMs: 300_000 } // 1分フレッシュ, 5分間はstaleで返す
  );
}
```

---

## まとめ

Claude Codeでキャッシュスタンピード対策を設計する：

1. **CLAUDE.md** にPER・Mutex・SWRの3パターンを用途別に明記し、TTLジッターを全キャッシュに適用
2. **PER（確率的早期更新）** はTTL残り時間に基づく確率でバックグラウンド更新——均等に負荷分散
3. **Mutex Lock** はNX付きのRedis SETで分散ロック——高コストクエリの同時実行を1つに絞る
4. **Stale-While-Revalidate** はsetImmediate()でバックグラウンド更新——レイテンシゼロで古いデータを返す

---

*キャッシュ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
