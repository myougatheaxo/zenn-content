---
title: "Claude CodeでRedisキャッシュを設計する：Cache-Aside・Write-Through・TTL戦略"
emoji: "🔴"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "cache"]
published: true
---

## はじめに

キャッシュなしのAPIは同じDBクエリを何度も実行する。Redisを正しく使うとレスポンスを10倍速くできるが、設計ミスするとデータ不整合が起きる。Claude Codeに安全な設計を生成させる。

---

## CLAUDE.mdにキャッシュルールを書く

```markdown
## Redisキャッシュ設計ルール

### パターン
- Cache-Aside: 読み取り多・書き込み少のデータ（ユーザープロフィール、商品マスタ）
- Write-Through: データ整合性が重要なデータ（残高、在庫）
- Pub/Sub: キャッシュ無効化の伝播（分散環境）

### TTL（必須）
- 全キャッシュにTTLを設定（永続キャッシュ禁止）
- マスタデータ: TTL 1時間
- セッション/プロフィール: TTL 15分
- 集計データ: TTL 5分

### キャッシュキー
- 形式: {サービス}:{エンティティ}:{ID}
- 例: user:profile:123, product:detail:abc
- バージョニング: user:v2:profile:123（スキーマ変更時にv番号を上げる）

### 注意
- キャッシュヒット率をメトリクスで監視
- Cold start（キャッシュ未暖機）を考慮した設計
```

---

## Cache-Aside パターンの生成

```
ユーザープロフィールのCache-Asideキャッシュを生成してください。

要件：
- キャッシュキー: user:profile:{userId}
- TTL: 15分
- キャッシュミス時はDBから取得してキャッシュに書き込む
- プロフィール更新時はキャッシュ削除
- キャッシュヒット率をログ出力

生成ファイル: src/services/userCacheService.ts
```

---

## 生成されるCache-Aside

```typescript
// src/services/userCacheService.ts
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
const CACHE_TTL = 15 * 60; // 15分
const KEY_PREFIX = 'user:profile';

function cacheKey(userId: string): string {
  return `${KEY_PREFIX}:${userId}`;
}

export async function getUserProfile(userId: string): Promise<UserProfile | null> {
  const key = cacheKey(userId);

  // キャッシュ確認
  const cached = await redis.get(key);
  if (cached) {
    logger.debug({ userId, cacheHit: true }, 'User profile cache hit');
    return JSON.parse(cached) as UserProfile;
  }

  logger.debug({ userId, cacheHit: false }, 'User profile cache miss');

  // DBから取得
  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: { id: true, name: true, email: true, avatarUrl: true, updatedAt: true },
  });

  if (!user) return null;

  // キャッシュに書き込み
  await redis.set(key, JSON.stringify(user), { EX: CACHE_TTL });

  return user;
}

export async function invalidateUserProfile(userId: string): Promise<void> {
  await redis.del(cacheKey(userId));
  logger.info({ userId }, 'User profile cache invalidated');
}
```

```typescript
// プロフィール更新時にキャッシュを削除
export async function updateUserProfile(
  userId: string,
  data: UpdateProfileInput
): Promise<UserProfile> {
  const updated = await prisma.user.update({
    where: { id: userId },
    data,
  });

  // 更新後にキャッシュ削除（次回アクセス時に再生成）
  await invalidateUserProfile(userId);

  return updated;
}
```

---

## Write-Through パターン

```typescript
// src/services/inventoryCacheService.ts
// 在庫データ: 整合性重視 → Write-Through

const INVENTORY_TTL = 5 * 60; // 5分

export async function updateInventory(
  productId: string,
  quantity: number
): Promise<void> {
  // DB書き込みとキャッシュ書き込みをトランザクションで（DBトランザクション + Redis pipeline）
  await prisma.$transaction(async (tx) => {
    await tx.inventory.update({
      where: { productId },
      data: { quantity },
    });
  });

  // DB成功後にキャッシュも更新（Write-Through）
  await redis.set(
    `inventory:${productId}`,
    JSON.stringify({ quantity, updatedAt: new Date() }),
    { EX: INVENTORY_TTL }
  );
}

export async function getInventory(productId: string): Promise<number> {
  const cached = await redis.get(`inventory:${productId}`);
  if (cached) return JSON.parse(cached).quantity;

  const inventory = await prisma.inventory.findUnique({ where: { productId } });
  return inventory?.quantity ?? 0;
}
```

---

## 分散キャッシュ無効化（Pub/Sub）

```typescript
// 複数サーバーでキャッシュを同期する場合
const publisher = createClient({ url: process.env.REDIS_URL });
const subscriber = publisher.duplicate();

// キャッシュ無効化メッセージを発行
async function publishCacheInvalidation(channel: string, key: string): Promise<void> {
  await publisher.publish(channel, JSON.stringify({ key, timestamp: Date.now() }));
}

// 全サーバーで購読してローカルキャッシュを削除
async function subscribeCacheInvalidation(): Promise<void> {
  await subscriber.subscribe('cache:invalidate', async (message) => {
    const { key } = JSON.parse(message);
    await redis.del(key);
    logger.info({ key }, 'Cache invalidated via pub/sub');
  });
}
```

---

## まとめ

Claude CodeでRedisキャッシュを設計する：

1. **CLAUDE.md** にTTL必須・キーフォーマット・パターン選定基準を明記
2. **Cache-Aside** で読み取り負荷を軽減（更新時はキャッシュ削除）
3. **Write-Through** でDB書き込みと同時にキャッシュも更新
4. **Pub/Sub** で複数サーバー間のキャッシュ無効化を伝播

---

*キャッシュ実装のレビュー（TTL未設定、キャッシュ汚染リスク、整合性問題）は **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
