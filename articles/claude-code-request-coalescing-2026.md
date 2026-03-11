---
title: "Claude Codeでリクエストコアレッシングを設計する：同一クエリの重複排除・キャッシュスタンピード対策"
emoji: "🔗"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "performance"]
published: true
published_at: "2026-03-15 23:00"
---

## はじめに

「同じAPIに同時に100リクエストが来てDBに100回同じクエリが走る」——リクエストコアレッシング（Request Coalescing）で同一クエリを1回にまとめ、結果を全リクエスターに返す設計をClaude Codeに生成させる。

---

## CLAUDE.mdにリクエストコアレッシング設計ルールを書く

```markdown
## リクエストコアレッシング設計ルール

### 適用条件
- 同一キーに対して複数の同時リクエストが予想されるエンドポイント
- キャッシュミス時のDB/外部API呼び出しが重い処理
- 結果が副作用なしで共有できる読み取りクエリ

### 実装方式
- インメモリMap + Promise: 同一プロセス内の重複排除
- Redis Lock + Pub/Sub: 複数サーバーインスタンス間の重複排除
- DataLoader: GraphQL/バッチ読み取り向けN+1排除

### コアレッシング対象外
- 書き込み操作（副作用あり）
- ユーザー固有の認証が必要なクエリ（結果を他ユーザーと共有できない）
- TTLが非常に短い（100ms未満）リアルタイムデータ
```

---

## リクエストコアレッシング実装の生成

```
リクエストコアレッシングで重複DB/APIアクセスを排除してください。

要件：
- インプロセスコアレッシング（同一Node.jsプロセス内）
- 分散コアレッシング（Redis経由、複数サーバー）
- タイムアウト付き
- メトリクス記録

生成ファイル: src/performance/coalescing/
```

---

## 生成されるリクエストコアレッシング実装

```typescript
// src/performance/coalescing/inProcessCoalescer.ts — インプロセスコアレッシング

export class InProcessCoalescer<T> {
  // 進行中のリクエストを保持するMap（key → Promise）
  private readonly inflight = new Map<string, Promise<T>>();

  async dedupe(
    key: string,
    fn: () => Promise<T>,
    options?: { timeoutMs?: number }
  ): Promise<T> {
    // 同一キーの進行中リクエストがあれば、そのPromiseを共有
    const existing = this.inflight.get(key);
    if (existing) {
      metrics.counter('coalescing_saved_calls', 1, { key });
      return existing;
    }

    // 新規リクエスト: Promiseを生成してMapに登録
    const promise = this.executeWithTimeout(fn, options?.timeoutMs ?? 30_000);

    this.inflight.set(key, promise);

    try {
      const result = await promise;
      return result;
    } finally {
      // 完了後はMapから削除（次回は新しいリクエストを開始）
      this.inflight.delete(key);
    }
  }

  private async executeWithTimeout<T>(fn: () => Promise<T>, timeoutMs: number): Promise<T> {
    return Promise.race([
      fn(),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new TimeoutError(`Coalescing timeout: ${timeoutMs}ms`)), timeoutMs)
      ),
    ]);
  }

  get inflightCount(): number {
    return this.inflight.size;
  }
}
```

```typescript
// src/performance/coalescing/distributedCoalescer.ts — 分散コアレッシング（Redis）

export class DistributedCoalescer<T> {
  private readonly localCoalescer = new InProcessCoalescer<T>();

  constructor(
    private readonly lockPrefix = 'coalesce:lock',
    private readonly resultPrefix = 'coalesce:result',
    private readonly lockTtlMs = 10_000,
    private readonly resultTtlMs = 30_000,
  ) {}

  async dedupe(
    key: string,
    fn: () => Promise<T>,
    cacheKey?: string
  ): Promise<T> {
    const lockKey = `${this.lockPrefix}:${key}`;
    const resultKey = cacheKey ?? `${this.resultPrefix}:${key}`;

    // まずローカルコアレッシング（同プロセス内の重複排除）
    return this.localCoalescer.dedupe(key, async () => {
      // 1. キャッシュ確認
      const cached = await redis.get(resultKey);
      if (cached) return JSON.parse(cached) as T;

      // 2. 分散ロック取得（SET NX PX）
      const lockAcquired = await redis.set(lockKey, '1', { NX: true, PX: this.lockTtlMs });

      if (lockAcquired) {
        // ロック取得成功: 自分が計算する
        try {
          const result = await fn();
          // 結果をキャッシュに保存（他のウェイターも参照できる）
          await redis.set(resultKey, JSON.stringify(result), { PX: this.resultTtlMs });
          return result;
        } finally {
          await redis.del(lockKey); // ロック解放 + Pub/Sub通知
          await redis.publish(`${lockKey}:done`, '1');
        }
      } else {
        // ロック取得失敗: 先行リクエストの完了を待機
        return this.waitForResult(resultKey, lockKey);
      }
    });
  }

  private async waitForResult(resultKey: string, lockKey: string): Promise<T> {
    const subscriber = createSubscriberConnection();
    const channel = `${lockKey}:done`;

    return new Promise<T>((resolve, reject) => {
      const timeout = setTimeout(async () => {
        await subscriber.unsubscribe(channel);
        subscriber.disconnect();
        reject(new TimeoutError('Distributed coalescing wait timeout'));
      }, this.lockTtlMs + 2000);

      subscriber.subscribe(channel);
      subscriber.on('message', async () => {
        clearTimeout(timeout);
        await subscriber.unsubscribe(channel);
        subscriber.disconnect();

        const cached = await redis.get(resultKey);
        if (cached) {
          resolve(JSON.parse(cached) as T);
        } else {
          reject(new Error('Result not found after lock release'));
        }
      });
    });
  }
}
```

```typescript
// src/performance/coalescing/userProfileCoalescer.ts — 実際の使用例

const profileCoalescer = new DistributedCoalescer<UserProfile>();

export async function getUserProfile(userId: string): Promise<UserProfile> {
  return profileCoalescer.dedupe(
    `user-profile:${userId}`,
    async () => {
      // この処理は同時多発リクエストがあっても1回だけ実行される
      const user = await prisma.user.findUniqueOrThrow({
        where: { id: userId },
        include: { subscription: true, settings: true },
      });

      return {
        id: user.id,
        name: user.name,
        email: user.email,
        plan: user.subscription?.plan ?? 'free',
        settings: user.settings,
      };
    },
    `cache:user-profile:${userId}` // Redisキャッシュキー
  );
}

// バッチコアレッシング（DataLoader方式）
export class BatchCoalescer<K, V> {
  private pending: Array<{ key: K; resolve: (v: V) => void; reject: (e: Error) => void }> = [];
  private timer: NodeJS.Timeout | null = null;

  constructor(
    private readonly batchFn: (keys: K[]) => Promise<Map<K, V>>,
    private readonly delayMs = 1 // 1msのバッチウィンドウ
  ) {}

  async load(key: K): Promise<V> {
    return new Promise<V>((resolve, reject) => {
      this.pending.push({ key, resolve, reject });

      if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.delayMs);
      }
    });
  }

  private async flush(): Promise<void> {
    this.timer = null;
    const batch = this.pending.splice(0);
    const keys = [...new Set(batch.map(b => b.key))];

    try {
      const results = await this.batchFn(keys);
      for (const { key, resolve, reject } of batch) {
        const value = results.get(key);
        if (value !== undefined) {
          resolve(value);
        } else {
          reject(new NotFoundError(`Key not found: ${String(key)}`));
        }
      }
    } catch (error) {
      batch.forEach(({ reject }) => reject(error as Error));
    }
  }
}

// ユーザー情報バッチローダー（N+1防止）
const userBatchLoader = new BatchCoalescer<string, User>(async (ids) => {
  const users = await prisma.user.findMany({ where: { id: { in: ids } } });
  return new Map(users.map(u => [u.id, u]));
});

// GraphQLリゾルバーでの使用
const resolvers = {
  Order: {
    user: (order: Order) => userBatchLoader.load(order.userId), // N+1が自動的にバッチ化される
  },
};
```

---

## まとめ

Claude Codeでリクエストコアレッシングを設計する：

1. **CLAUDE.md** に適用条件（読み取り・副作用なし）・インプロセス/分散の使い分け・コアレッシング対象外の明示を記載
2. **InProcessCoalescer** でMap+Promiseの共有——同一プロセス内なら追加依存なしで実装可能、コードが最もシンプル
3. **DistributedCoalescer** でRedis SET NX + Pub/Sub——マルチサーバー環境でも1台だけ計算し結果を全サーバーに配布
4. **BatchCoalescer（DataLoader方式）** で1msウィンドウ内の複数loadを1つのIN句クエリに——GraphQLのN+1問題を透過的に解決

---

*パフォーマンス設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
