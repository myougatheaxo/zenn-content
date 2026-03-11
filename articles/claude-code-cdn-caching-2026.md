---
title: "Claude CodeでCDNキャッシュを設計する：Cache-Control・Surrogate-Key・Edge Side Includes"
emoji: "🌐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "cdn", "performance"]
published: true
---

## はじめに

APIレスポンスのキャッシュをCDNに委ねると、オリジンサーバーへのトラフィックを90%削減できる。しかしCache-Controlの設定ミスでユーザーに古いデータが届く。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにCDNキャッシュ設計ルールを書く

```markdown
## CDNキャッシュ設計ルール

### Cache-Control ヘッダー戦略
- 公開・不変コンテンツ: public, max-age=31536000, immutable
- 公開・更新あり: public, max-age=60, stale-while-revalidate=300
- 認証済みユーザー: private, no-store（CDNにキャッシュさせない）
- APIレスポンス（認証不要）: public, max-age=30, s-maxage=300

### パージ戦略
- Surrogate-Key（Fastly）/ Cache-Tag（Cloudflare）でタグベースパージ
- エンティティ更新時に関連するすべてのキャッシュを一括パージ
- 全パージ（purge all）は緊急時のみ

### バリエーション
- Accept-Encoding を Vary に含める（gzip/brotli対応）
- 認証済みかどうかでコンテンツが変わる場合は Vary: Cookie
- Accept-Language で多言語コンテンツを分離
```

---

## CDNキャッシュ最適化の生成

```
CDNキャッシュ最適化システムを設計してください。

要件：
- レスポンスにCache-Controlヘッダーを付与
- Surrogate-Keyでタグベースキャッシュパージ
- コンテンツ種別（公開/認証/動的）に応じたキャッシュ戦略
- Fastly / Cloudflare対応

生成ファイル: src/middleware/cacheControl.ts
```

---

## 生成されるCDNキャッシュ実装

```typescript
// src/middleware/cacheControl.ts

type CacheStrategy =
  | 'public-static'         // 画像・CSS・JS（不変）
  | 'public-short'          // 公開API（30秒）
  | 'public-medium'         // 公開コンテンツ（5分）
  | 'public-long'           // 公開コンテンツ（1時間）
  | 'private'               // 認証済みユーザー専用
  | 'no-cache';             // 常に最新を取得

const CACHE_HEADERS: Record<CacheStrategy, string> = {
  'public-static': 'public, max-age=31536000, immutable',
  'public-short': 'public, max-age=30, s-maxage=300, stale-while-revalidate=60',
  'public-medium': 'public, max-age=300, s-maxage=1800, stale-while-revalidate=600',
  'public-long': 'public, max-age=3600, s-maxage=86400, stale-while-revalidate=3600',
  'private': 'private, no-store',
  'no-cache': 'no-cache, no-store, must-revalidate',
};

export function setCacheControl(strategy: CacheStrategy) {
  return (_req: Request, res: Response, next: NextFunction) => {
    res.setHeader('Cache-Control', CACHE_HEADERS[strategy]);
    next();
  };
}

// Surrogate-Key（タグベースパージ用）
export function addSurrogateKeys(keys: string[]) {
  return (_req: Request, res: Response, next: NextFunction) => {
    // Fastly: Surrogate-Key / Cloudflare: Cache-Tag
    res.setHeader('Surrogate-Key', keys.join(' '));
    res.setHeader('Cache-Tag', keys.join(','));
    next();
  };
}
```

```typescript
// src/routes/products.ts
// 商品一覧: 公開・5分キャッシュ + タグベースパージ
router.get(
  '/products',
  setCacheControl('public-medium'),
  addSurrogateKeys(['products', 'product-list']),
  async (req, res) => {
    const products = await prisma.product.findMany({
      where: { published: true },
      orderBy: { createdAt: 'desc' },
    });
    res.json(products);
  }
);

// 商品詳細: 商品IDタグを付与（その商品のキャッシュだけパージ可能）
router.get(
  '/products/:id',
  setCacheControl('public-medium'),
  async (req, res, next) => {
    addSurrogateKeys([`products`, `product:${req.params.id}`])(req, res, () => {});
    next();
  },
  async (req, res) => {
    const product = await prisma.product.findUniqueOrThrow({
      where: { id: req.params.id },
    });
    res.json(product);
  }
);

// 認証済みユーザーのプロフィール: CDNキャッシュしない
router.get('/me', authenticate, setCacheControl('private'), async (req, res) => {
  const user = await prisma.user.findUniqueOrThrow({
    where: { id: req.user.id },
    select: { id: true, name: true, email: true },
  });
  res.json(user);
});
```

---

## タグベースキャッシュパージ

```typescript
// src/services/cacheInvalidation.ts

// Fastly APIでSurrogate-Keyパージ
async function purgeFastlyByTag(tag: string): Promise<void> {
  const response = await fetch(
    `https://api.fastly.com/service/${process.env.FASTLY_SERVICE_ID}/purge/${tag}`,
    {
      method: 'POST',
      headers: {
        'Fastly-Key': process.env.FASTLY_API_KEY!,
        'Accept': 'application/json',
      },
    }
  );

  if (!response.ok) {
    throw new Error(`Fastly purge failed: ${response.status}`);
  }

  logger.info({ tag }, 'Fastly cache purged');
}

// Cloudflare Cache-Tagパージ
async function purgeCloudflareByTag(tag: string): Promise<void> {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${process.env.CF_ZONE_ID}/purge_cache`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.CF_API_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ tags: [tag] }),
    }
  );

  const data = await response.json();
  if (!data.success) {
    throw new Error(`Cloudflare purge failed: ${JSON.stringify(data.errors)}`);
  }

  logger.info({ tag }, 'Cloudflare cache purged');
}

// 商品更新後に関連キャッシュをすべてパージ
export async function invalidateProductCache(productId: string): Promise<void> {
  await Promise.allSettled([
    purgeFastlyByTag(`product:${productId}`),
    purgeFastlyByTag('product-list'), // 一覧も更新
  ]);
}
```

---

## Prisma middleware でDB更新時に自動パージ

```typescript
// src/lib/prisma.ts
prisma.$use(async (params, next) => {
  const result = await next(params);

  // 書き込み操作後にキャッシュパージ
  if (['create', 'update', 'delete', 'upsert'].includes(params.action)) {
    if (params.model === 'Product') {
      const productId = params.args.where?.id ?? result?.id;
      if (productId) {
        // 非同期でパージ（レスポンスをブロックしない）
        setImmediate(() => {
          invalidateProductCache(productId).catch(err =>
            logger.error({ err }, 'Cache invalidation failed')
          );
        });
      }
    }
  }

  return result;
});
```

---

## まとめ

Claude CodeでCDNキャッシュを設計する：

1. **CLAUDE.md** にコンテンツ種別ごとのCache-Control戦略・パージ方針を明記
2. **s-maxage** でCDNのTTLとブラウザのTTLを分離（CDNは長く、ブラウザは短く）
3. **Surrogate-Key / Cache-Tag** でエンティティIDタグを付与（ピンポイントパージ）
4. **Prisma middleware** でDB更新時に自動的にキャッシュをパージ

---

*CDNキャッシュ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
