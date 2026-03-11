---
title: "React Server Componentsパターン：Claude Codeで実装するRSC設計"
emoji: "⚛️"
type: "tech"
topics:
  - claudecode
  - react
  - nextjs
  - typescript
published: true
---

## React Server Componentsとは何か

React Server Components（RSC）は、コンポーネントをサーバー側で実行し、シリアライズされた結果をクライアントに送るパラダイムだ。Next.js 13以降のApp Routerではデフォルトでサーバーコンポーネントになる。

RSCの最大のメリットはバンドルサイズの削減だ。サーバーでのみ使われるライブラリはクライアントのJSに含まれない。データフェッチもサーバー直接実行できるため、ウォーターフォールを排除できる。

しかし「どこがServer/Clientか」の境界設計を間違えると、意図しないハイドレーションエラーや、クライアントコードへの機密情報漏洩につながる。

## Server/Clientの境界設計

```typescript
// app/products/page.tsx - Server Component（デフォルト）
// この中でasync/awaitが使える。クライアントに含まれない
import { db } from "@/lib/db";
import { ProductCard } from "./ProductCard";
import { AddToCartButton } from "./AddToCartButton";

export default async function ProductsPage() {
  // サーバーで直接DBアクセス - API routeが不要
  const products = await db.product.findMany({
    orderBy: { createdAt: "desc" },
    take: 20,
  });

  return (
    <div className="grid grid-cols-3 gap-4">
      {products.map((product) => (
        <div key={product.id}>
          {/* Server Component: DBデータを直接渡す */}
          <ProductCard product={product} />
          {/* Client Component: インタラクションが必要 */}
          <AddToCartButton productId={product.id} price={product.price} />
        </div>
      ))}
    </div>
  );
}
```

```typescript
// app/products/AddToCartButton.tsx - Client Component
// インタラクションが必要な部分だけ "use client" にする
"use client";

import { useState } from "react";
import { addToCart } from "@/app/actions";

interface Props {
  productId: string;
  price: number;
}

export function AddToCartButton({ productId, price }: Props) {
  const [loading, setLoading] = useState(false);
  const [added, setAdded] = useState(false);

  const handleClick = async () => {
    setLoading(true);
    await addToCart(productId);
    setAdded(true);
    setLoading(false);
  };

  return (
    <button
      onClick={handleClick}
      disabled={loading || added}
      className="btn-primary"
    >
      {loading ? "追加中..." : added ? "カートに追加済み" : `¥${price.toLocaleString()} カートへ`}
    </button>
  );
}
```

## Server Actionsとデータ変更

```typescript
// app/actions.ts - Server Actions
"use server";

import { db } from "@/lib/db";
import { revalidatePath } from "next/cache";
import { auth } from "@/lib/auth";
import { z } from "zod";

const AddToCartSchema = z.object({
  productId: z.string().cuid(),
  quantity: z.number().int().positive().default(1),
});

export async function addToCart(productId: string, quantity = 1): Promise<void> {
  // Server Actionの中でセッション確認ができる
  const session = await auth();
  if (!session?.user?.id) {
    throw new Error("ログインが必要です");
  }

  // バリデーション
  const validated = AddToCartSchema.parse({ productId, quantity });

  await db.cartItem.upsert({
    where: {
      userId_productId: {
        userId: session.user.id,
        productId: validated.productId,
      },
    },
    create: {
      userId: session.user.id,
      productId: validated.productId,
      quantity: validated.quantity,
    },
    update: {
      quantity: { increment: validated.quantity },
    },
  });

  // キャッシュを無効化してUIを更新
  revalidatePath("/cart");
}

export async function updateCartQuantity(
  cartItemId: string,
  quantity: number,
): Promise<{ success: boolean; error?: string }> {
  const session = await auth();
  if (!session?.user?.id) {
    return { success: false, error: "Unauthorized" };
  }

  if (quantity <= 0) {
    await db.cartItem.delete({ where: { id: cartItemId } });
  } else {
    await db.cartItem.update({
      where: { id: cartItemId, userId: session.user.id },
      data: { quantity },
    });
  }

  revalidatePath("/cart");
  return { success: true };
}
```

## データフェッチのパターン：並列・逐次・ストリーミング

```typescript
// app/dashboard/page.tsx
import { Suspense } from "react";
import { getUserStats, getRecentOrders, getRecommendations } from "@/lib/api";

// 並列フェッチ（Promise.allで複数を同時取得）
export default async function DashboardPage() {
  // 独立したデータを並列で取得
  const [stats, recentOrders] = await Promise.all([
    getUserStats(),
    getRecentOrders({ limit: 5 }),
  ]);

  return (
    <div>
      <StatsPanel stats={stats} />
      <RecentOrders orders={recentOrders} />
      {/* 重いデータはSuspenseで遅延ロード */}
      <Suspense fallback={<RecommendationsSkeleton />}>
        <RecommendationsSection />
      </Suspense>
    </div>
  );
}

// RecommendationsSection は独立したServer Componentとして定義
// ページ全体をブロックせずに遅延ロードされる
async function RecommendationsSection() {
  const recommendations = await getRecommendations();
  return <RecommendationsList items={recommendations} />;
}
```

## キャッシュ戦略

```typescript
import { unstable_cache as cache } from "next/cache";

// Next.jsのキャッシュAPIで細かく制御
const getPopularProducts = cache(
  async (categoryId: string) => {
    return await db.product.findMany({
      where: { categoryId, isPublished: true },
      orderBy: { salesCount: "desc" },
      take: 10,
    });
  },
  ["popular-products"],  // キャッシュキー
  {
    revalidate: 60 * 60,  // 1時間キャッシュ
    tags: ["products"],   // タグで一括無効化
  },
);

// 特定のタグのキャッシュを無効化（Server Actionから）
import { revalidateTag } from "next/cache";

export async function publishProduct(productId: string) {
  await db.product.update({
    where: { id: productId },
    data: { isPublished: true },
  });
  // productsタグのキャッシュをすべて無効化
  revalidateTag("products");
}

// fetch()のキャッシュオプション
async function fetchExternalData() {
  const res = await fetch("https://api.example.com/data", {
    next: {
      revalidate: 3600,  // 1時間キャッシュ
      tags: ["external-data"],
    },
  });
  return res.json();
}
```

## エラーハンドリングとローディング状態

```typescript
// app/products/error.tsx - エラーバウンダリ
"use client";

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function ProductsError({ error, reset }: ErrorProps) {
  return (
    <div className="error-container">
      <h2>商品の読み込みに失敗しました</h2>
      <p className="text-gray-500">{error.message}</p>
      <button onClick={reset} className="btn-secondary">
        再試行
      </button>
    </div>
  );
}

// app/products/loading.tsx - ローディングUI
export default function ProductsLoading() {
  return (
    <div className="grid grid-cols-3 gap-4">
      {Array.from({ length: 6 }).map((_, i) => (
        <div key={i} className="skeleton-card animate-pulse">
          <div className="h-48 bg-gray-200 rounded" />
          <div className="h-4 bg-gray-200 rounded mt-2" />
          <div className="h-4 bg-gray-200 rounded mt-1 w-2/3" />
        </div>
      ))}
    </div>
  );
}
```

RSCはServer/Clientの境界を意識した設計が必要だが、一度パターンを掴めばデータフェッチの複雑さが大幅に減る。「インタラクションが必要な最小単位だけClientにする」という原則を守ることが設計の核心だ。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
