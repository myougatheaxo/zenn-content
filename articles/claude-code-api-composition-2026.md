---
title: "Claude CodeでAPIコンポジションを設計する：BFF・複数サービス集約・データ変換"
emoji: "🔗"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 10:00"
---

## はじめに

「フロントエンドが複数のマイクロサービスを直接叩いていてAPIコールが多い」——BFF（Backend for Frontend）でAPIコンポジションレイヤーを設け、複数サービスのデータを1つのAPIレスポンスに集約する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにAPIコンポジション設計ルールを書く

```markdown
## APIコンポジション設計ルール

### コンポジション原則
- BFFが複数のダウンストリームサービスを並列で呼び出す
- フロントエンドに必要なデータだけを返す（過剰データを除去）
- エラーはサービス単位で分離（1サービス失敗でもレスポンスを返す）

### パフォーマンス
- 独立したサービス呼び出しは常に並列実行（Promise.all）
- 依存関係がある場合のみ逐次（例: userId取得後にそのユーザーの注文取得）
- 結果をBFFレベルでキャッシュ（TTL: 30秒）

### データ変換
- ダウンストリームのレスポンスをBFFで変換（フロントの型に合わせる）
- camelCase変換、フィールド名の統一、不要フィールドの除去
```

---

## APIコンポジション実装の生成

```
BFF APIコンポジションを設計してください。

要件：
- 複数サービスの並列呼び出し
- 部分的な失敗の処理
- レスポンスデータ変換
- BFFキャッシュ

生成ファイル: src/bff/composition/
```

---

## 生成されるAPIコンポジション実装

```typescript
// src/bff/composition/composer.ts — APIコンポジションエンジン

export interface DataSource<T> {
  name: string;
  fetch: () => Promise<T>;
  optional?: boolean;      // true: 失敗してもレスポンスに含めない（エラーにしない）
  cacheTtlSec?: number;    // BFFレベルキャッシュのTTL
  fallback?: () => T;      // フォールバック値
}

export type ComposedResult<T extends Record<string, DataSource<unknown>>> = {
  [K in keyof T]: T[K] extends DataSource<infer U> ? U | null : never;
};

export class ApiComposer {
  async compose<T extends Record<string, DataSource<unknown>>>(
    sources: T,
    options: { cacheKey?: string; cacheTtlSec?: number } = {}
  ): Promise<{ data: ComposedResult<T>; partial: boolean }> {
    // BFFレベルのキャッシュチェック
    if (options.cacheKey) {
      const cached = await redis.get(`bff:composed:${options.cacheKey}`);
      if (cached) {
        return { data: JSON.parse(cached), partial: false };
      }
    }

    // 全ソースを並列で取得
    const entries = Object.entries(sources) as Array<[string, DataSource<unknown>]>;

    const results = await Promise.allSettled(
      entries.map(([name, source]) =>
        this.fetchWithCache(name, source)
      )
    );

    const data: Record<string, unknown> = {};
    let partial = false;

    for (let i = 0; i < entries.length; i++) {
      const [key, source] = entries[i];
      const result = results[i];

      if (result.status === 'fulfilled') {
        data[key] = result.value;
      } else {
        if (source.optional) {
          // オプショナルソースの失敗: nullを設定して続行
          data[key] = source.fallback ? source.fallback() : null;
          partial = true;
          logger.warn({ source: source.name, error: result.reason.message }, 'Optional source failed');
        } else {
          // 必須ソースの失敗: エラーをスロー
          throw new CompositionError(`Required source '${source.name}' failed: ${result.reason.message}`);
        }
      }
    }

    const composedResult = data as ComposedResult<T>;

    // BFFレベルキャッシュに保存
    if (options.cacheKey && !partial) {
      const ttl = options.cacheTtlSec ?? 30;
      await redis.set(`bff:composed:${options.cacheKey}`, JSON.stringify(composedResult), { EX: ttl });
    }

    return { data: composedResult, partial };
  }

  private async fetchWithCache<T>(name: string, source: DataSource<T>): Promise<T> {
    if (!source.cacheTtlSec) return source.fetch();

    const cacheKey = `bff:source:${name}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached) as T;

    const result = await source.fetch();
    await redis.set(cacheKey, JSON.stringify(result), { EX: source.cacheTtlSec });
    return result;
  }
}
```

```typescript
// src/bff/composition/orderDetailComposer.ts — 注文詳細ページのAPIコンポジション

export class OrderDetailComposer {
  private readonly composer = new ApiComposer();

  async getOrderDetail(orderId: string, userId: string): Promise<OrderDetailResponse> {
    const { data, partial } = await this.composer.compose({
      // 必須: 注文情報
      order: {
        name: 'order-service',
        fetch: () => orderServiceClient.getOrder(orderId),
      },

      // 必須: ユーザー情報
      user: {
        name: 'user-service',
        fetch: () => userServiceClient.getUser(userId),
        cacheTtlSec: 300,  // ユーザー情報は5分キャッシュ
      },

      // オプショナル: 配送状況（追跡API遅い場合もある）
      shipping: {
        name: 'shipping-service',
        fetch: () => shippingServiceClient.getStatus(orderId),
        optional: true,
        fallback: () => ({ status: 'unknown', estimatedDelivery: null }),
      },

      // オプショナル: レビュー（購入済み商品のレビュー）
      review: {
        name: 'review-service',
        fetch: () => reviewServiceClient.getUserReviewForOrder(userId, orderId),
        optional: true,
        cacheTtlSec: 60,
      },
    }, {
      cacheKey: `order-detail:${orderId}:${userId}`,
      cacheTtlSec: 30,
    });

    // レスポンスをフロントエンドの型に変換
    return this.transform(data, partial);
  }

  private transform(
    data: { order: Order; user: User; shipping: ShippingStatus | null; review: Review | null },
    partial: boolean
  ): OrderDetailResponse {
    return {
      orderId: data.order.id,
      status: data.order.status,
      placedAt: data.order.createdAt,
      totalAmount: data.order.totalAmount,
      currency: data.order.currency,

      customer: {
        name: data.user.displayName,
        email: data.user.email,
        avatarUrl: data.user.profileImageUrl,
      },

      items: data.order.items.map(item => ({
        productId: item.productId,
        productName: item.name,
        quantity: item.quantity,
        unitPrice: item.price,
        subtotal: item.price * item.quantity,
      })),

      shipping: data.shipping ? {
        status: data.shipping.status,
        trackingNumber: data.shipping.trackingNumber,
        estimatedDelivery: data.shipping.estimatedDelivery,
        carrier: data.shipping.carrier,
      } : null,

      review: data.review ? {
        rating: data.review.rating,
        comment: data.review.comment,
        reviewedAt: data.review.createdAt,
      } : null,

      _meta: { partial },
    };
  }
}

// API エンドポイント
router.get('/api/bff/orders/:orderId', requireAuth, async (req, res) => {
  const composer = new OrderDetailComposer();
  const result = await composer.getOrderDetail(req.params.orderId, req.user.id);

  if (result._meta.partial) {
    res.set('X-Partial-Response', 'true');
  }

  res.json(result);
});

// 依存関係がある場合の逐次コンポジション
router.get('/api/bff/users/:username/feed', requireAuth, async (req, res) => {
  // Step 1: usernameからuserIdを解決
  const user = await userServiceClient.getUserByUsername(req.params.username);

  // Step 2: userId依存の並列取得
  const composer = new ApiComposer();
  const { data } = await composer.compose({
    posts: { name: 'post-service', fetch: () => postServiceClient.getUserPosts(user.id, { limit: 20 }) },
    followers: { name: 'social-service', fetch: () => socialServiceClient.getFollowerCount(user.id), cacheTtlSec: 300, optional: true },
    isFollowing: { name: 'social-service-follow', fetch: () => socialServiceClient.isFollowing(req.user.id, user.id), optional: true },
  });

  res.json({ user, posts: data.posts, followerCount: data.followers ?? 0, isFollowing: data.isFollowing ?? false });
});
```

---

## まとめ

Claude CodeでAPIコンポジションを設計する：

1. **CLAUDE.md** にBFFで複数サービスを並列呼び出し・オプショナルソース失敗はnull返し（エラーにしない）・BFFレベルキャッシュ30秒・依存関係がある場合のみ逐次を明記
2. **`optional: true` + `fallback`** でオプショナルサービスの失敗を分離——配送状況APIが遅くても注文詳細は返せる。`X-Partial-Response: true`ヘッダーでフロントに部分レスポンスを通知
3. **BFFレベルのデータ変換** でダウンストリームの型からフロントエンドの型へ変換——`profileImageUrl` → `avatarUrl`など命名の統一、不要フィールドの除去をここで吸収
4. **2段階コンポジション** で依存関係を解決——`username → userId`（逐次）後に「そのuserIdに依存する全データ」を並列取得。逐次の後に並列を置いて最小限の待機時間を実現

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
