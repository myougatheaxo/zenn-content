---
title: "Claude CodeでCQRSパターンを設計する：コマンド・クエリ分離・読み取りモデル最適化"
emoji: "📊"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "redis"]
published: true
published_at: "2026-03-15 17:00"
---

## はじめに

「同じDBモデルで書き込みも読み込みも捌く」——それが複雑なビジネスロジックとパフォーマンス要件を同時に満たせなくなる根本原因。CQRSでコマンド（書き込み）とクエリ（読み込み）を分離する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにCQRS設計ルールを書く

```markdown
## CQRS設計ルール

### コマンド（書き込み側）
- ドメインモデル（集約）でビジネスロジックを表現
- コマンドハンドラーはDB書き込み + イベント発行のみ
- バリデーション: コマンドクラスのconstructorで完結

### クエリ（読み込み側）
- 読み取りモデルはコマンドモデルと完全分離
- DBビューまたはRedisから直接取得（Prismaモデル不使用）
- N+1が発生する場合は読み取りモデル自体を変更（JOIN済みビュー等）

### イベント連携
- コマンド完了後にドメインイベントをRedis Streamsに発行
- 読み取りモデルを更新するProjectorがイベントを購読・反映
```

---

## CQRS実装の生成

```
CQRSパターンを実装してください。

要件：
- コマンドハンドラー（書き込み）
- クエリハンドラー（読み込み）
- ドメインイベント発行
- 読み取りモデルのProjector

生成ファイル: src/cqrs/
```

---

## 生成されるCQRS実装

```typescript
// src/cqrs/commands/order/PlaceOrderCommand.ts — コマンド定義

export class PlaceOrderCommand {
  readonly userId: string;
  readonly items: Array<{ productId: string; quantity: number }>;
  readonly shippingAddressId: string;

  constructor(input: unknown) {
    const parsed = PlaceOrderSchema.parse(input); // Zodバリデーション
    this.userId = parsed.userId;
    this.items = parsed.items;
    this.shippingAddressId = parsed.shippingAddressId;
  }
}

// src/cqrs/commands/order/PlaceOrderHandler.ts — コマンドハンドラー

export class PlaceOrderHandler {
  async handle(command: PlaceOrderCommand): Promise<{ orderId: string }> {
    // ドメインロジック: 在庫確認・価格計算
    const products = await prisma.product.findMany({
      where: { id: { in: command.items.map(i => i.productId) } },
    });

    const orderItems = command.items.map(item => {
      const product = products.find(p => p.id === item.productId);
      if (!product) throw new NotFoundError(`Product ${item.productId} not found`);
      if (product.inventory < item.quantity) throw new ValidationError('Insufficient inventory');
      return { productId: item.productId, quantity: item.quantity, unitPrice: product.price };
    });

    const totalAmount = orderItems.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0);

    // トランザクション: 注文作成 + 在庫減少
    const order = await prisma.$transaction(async (tx) => {
      const created = await tx.order.create({
        data: {
          userId: command.userId,
          status: 'confirmed',
          totalAmount,
          shippingAddressId: command.shippingAddressId,
          items: { create: orderItems },
        },
      });

      // 在庫減少
      for (const item of orderItems) {
        await tx.product.update({
          where: { id: item.productId },
          data: { inventory: { decrement: item.quantity } },
        });
      }

      return created;
    });

    // ドメインイベント発行（非同期、コマンド完了後）
    await eventBus.publish('order.placed', {
      orderId: order.id,
      userId: command.userId,
      totalAmount,
      items: orderItems,
      placedAt: order.createdAt,
    });

    logger.info({ orderId: order.id, userId: command.userId }, 'Order placed');
    return { orderId: order.id };
  }
}
```

```typescript
// src/cqrs/queries/order/GetOrderListQuery.ts — クエリ（読み取りモデル）

export interface OrderListItem {
  orderId: string;
  status: string;
  totalAmount: number;
  itemCount: number;
  firstItemName: string; // 代表商品名
  placedAt: Date;
}

export class GetOrderListHandler {
  // 読み取りモデルはRedisから直接（コマンドモデルのPrismaは使わない）
  async handle(userId: string, page = 1, limit = 20): Promise<{
    items: OrderListItem[];
    total: number;
  }> {
    const cacheKey = `orders:list:${userId}:${page}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // 読み取り専用SQL（JOINで一発取得）
    const rows = await prisma.$queryRaw<Array<{
      order_id: string;
      status: string;
      total_amount: number;
      item_count: bigint;
      first_item_name: string;
      placed_at: Date;
      total_count: bigint;
    }>>`
      SELECT
        o.id AS order_id,
        o.status,
        o.total_amount,
        COUNT(oi.id) AS item_count,
        MIN(p.name) AS first_item_name,
        o.created_at AS placed_at,
        COUNT(*) OVER() AS total_count
      FROM orders o
      JOIN order_items oi ON oi.order_id = o.id
      JOIN products p ON p.id = oi.product_id
      WHERE o.user_id = ${userId}
        AND o.deleted_at IS NULL
      GROUP BY o.id, o.status, o.total_amount, o.created_at
      ORDER BY o.created_at DESC
      LIMIT ${limit} OFFSET ${(page - 1) * limit}
    `;

    const result = {
      items: rows.map(r => ({
        orderId: r.order_id,
        status: r.status,
        totalAmount: r.total_amount,
        itemCount: Number(r.item_count),
        firstItemName: r.first_item_name,
        placedAt: r.placed_at,
      })),
      total: rows.length > 0 ? Number(rows[0].total_count) : 0,
    };

    // 5分キャッシュ（読み取りモデルは多少古くてもよい）
    await redis.set(cacheKey, JSON.stringify(result), { EX: 300 });
    return result;
  }
}
```

```typescript
// src/cqrs/projectors/OrderReadModelProjector.ts — イベント→読み取りモデル更新

export class OrderReadModelProjector {
  async onOrderPlaced(event: OrderPlacedEvent): Promise<void> {
    // 読み取りモデルのキャッシュを無効化
    const pattern = `orders:list:${event.userId}:*`;
    const keys = await redis.keys(pattern);
    if (keys.length > 0) await redis.del(...keys);

    // 注文詳細をRedisに事前計算（次回クエリを高速化）
    const detail = await this.buildOrderDetail(event.orderId);
    await redis.set(`orders:detail:${event.orderId}`, JSON.stringify(detail), { EX: 3600 });

    logger.debug({ orderId: event.orderId }, 'OrderReadModel updated');
  }

  private async buildOrderDetail(orderId: string) {
    return prisma.$queryRaw`
      SELECT o.*, json_agg(
        json_build_object(
          'productId', oi.product_id,
          'quantity', oi.quantity,
          'unitPrice', oi.unit_price,
          'productName', p.name
        )
      ) AS items
      FROM orders o
      JOIN order_items oi ON oi.order_id = o.id
      JOIN products p ON p.id = oi.product_id
      WHERE o.id = ${orderId}
      GROUP BY o.id
    `;
  }
}

// src/cqrs/bus/CommandBus.ts — コマンドバス

type Handler<C, R> = { handle(command: C): Promise<R> };

export class CommandBus {
  private handlers = new Map<string, Handler<any, any>>();

  register<C, R>(commandClass: new (...args: any[]) => C, handler: Handler<C, R>): void {
    this.handlers.set(commandClass.name, handler);
  }

  async dispatch<R>(command: object): Promise<R> {
    const handler = this.handlers.get(command.constructor.name);
    if (!handler) throw new Error(`No handler for ${command.constructor.name}`);
    return handler.handle(command);
  }
}

// 使用例（Expressルーター）
router.post('/api/orders', requireAuth, async (req, res) => {
  const command = new PlaceOrderCommand({ ...req.body, userId: req.userId });
  const result = await commandBus.dispatch<{ orderId: string }>(command);
  res.status(201).json(result);
});

router.get('/api/orders', requireAuth, async (req, res) => {
  const handler = new GetOrderListHandler();
  const result = await handler.handle(req.userId, Number(req.query.page) || 1);
  res.json(result);
});
```

---

## まとめ

Claude CodeでCQRSパターンを設計する：

1. **CLAUDE.md** にコマンドはドメインロジック+イベント発行のみ・クエリはPrismaモデル不使用・N+1はビュー変更で解決を明記
2. **コマンドハンドラー** でバリデーション→トランザクション→イベント発行の責務を分離——読み書きを一切混在させない
3. **読み取りモデル** はJOIN済みSQL + Redisキャッシュで直接取得——ドメインモデルを介さないので複雑なビジネスロジックに影響されない
4. **Projector** がドメインイベントを購読して読み取りキャッシュを更新——書き込みと読み込みの同期は非同期でOK

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
