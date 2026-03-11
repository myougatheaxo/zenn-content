---
title: "Claude CodeでCQRS読み取りモデルを設計する：Projectionの構築・読み取り最適化・非同期同期"
emoji: "📖"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-20 11:00"
---

## はじめに

「書き込みモデルを直接クエリするとN+1問題やJOINが複雑になる」「ダッシュボード表示のためだけに集計クエリが重い」——CQRSの読み取り側でProjectionを構築し、読み取り専用の最適化されたモデルを持つ設計をClaude Codeに生成させる。

---

## CLAUDE.mdに読み取りモデル設計ルールを書く

```markdown
## CQRS読み取りモデル設計ルール

### 読み取りモデルの役割
- 書き込みモデル（集約）とは独立したデータ構造
- 表示・検索・集計に特化したスキーマ（非正規化可）
- ドメインイベントを受け取ってProjectionが更新する

### Projection設計
- イベント1件につきProjectionが1件の読み取りモデルを更新
- 再構築可能（全イベントを最初から再生すれば再構築できる）
- 冪等（同じイベントを2回処理しても結果が同じ）

### 同期戦略
- 同期: DBトランザクション内でProjectionも更新（強整合）
- 非同期: ドメインイベント経由でProjectionを更新（結果整合）
- 推奨: 非同期（書き込みパフォーマンスに影響しない）
```

---

## 読み取りモデル実装の生成

```
CQRS読み取りモデルのProjectionを設計してください。

要件：
- ドメインイベント駆動のProjection更新
- 非正規化された読み取り専用テーブル
- Projection再構築機能
- クエリサービス

生成ファイル: src/readmodel/
```

---

## 生成される読み取りモデル実装

```typescript
// src/readmodel/orderReadModel.ts — 読み取りモデルの型定義

// 書き込みモデル（集約）とは異なる非正規化された読み取りモデル
export interface OrderReadModel {
  // 注文基本情報
  orderId: string;
  userId: string;
  userEmail: string;     // Userから非正規化（JOINなしで取得可能）
  userDisplayName: string;

  // 注文詳細（アイテムを埋め込み）
  status: string;
  items: Array<{
    productId: string;
    productName: string;   // Productから非正規化
    quantity: number;
    unitPrice: number;
    subtotal: number;
  }>;

  // 集計値（計算済み）
  itemCount: number;
  totalAmount: number;
  currency: string;

  // タイムスタンプ
  createdAt: Date;
  submittedAt?: Date;
  completedAt?: Date;

  // 検索用インデックス列
  searchText: string;    // 全文検索用（orderId + userEmail + productNames）
}

// ダッシュボード用集計モデル
export interface UserOrderStats {
  userId: string;
  totalOrders: number;
  completedOrders: number;
  totalSpent: number;
  averageOrderValue: number;
  lastOrderAt?: Date;
  favoriteProductId?: string;
}
```

```typescript
// src/readmodel/projections/orderProjection.ts — Projection処理

export class OrderProjection {
  constructor(private readonly db: PrismaClient) {}

  // OrderCreatedイベント → 読み取りモデル作成
  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    // 非正規化に必要なデータを収集
    const [user, products] = await Promise.all([
      this.db.user.findUnique({ where: { id: event.userId } }),
      this.db.product.findMany({
        where: { id: { in: event.items.map(i => i.productId) } },
      }),
    ]);

    const productMap = new Map(products.map(p => [p.id, p]));

    const items = event.items.map(item => {
      const product = productMap.get(item.productId)!;
      return {
        productId: item.productId,
        productName: product.name,
        quantity: item.quantity,
        unitPrice: item.unitPrice,
        subtotal: item.quantity * item.unitPrice,
      };
    });

    const totalAmount = items.reduce((sum, i) => sum + i.subtotal, 0);
    const searchText = [
      event.orderId,
      user?.email,
      ...items.map(i => i.productName),
    ].filter(Boolean).join(' ').toLowerCase();

    await this.db.orderReadModel.upsert({
      where: { orderId: event.orderId },
      create: {
        orderId: event.orderId,
        userId: event.userId,
        userEmail: user?.email ?? '',
        userDisplayName: user?.displayName ?? '',
        status: 'draft',
        items: JSON.stringify(items),
        itemCount: items.length,
        totalAmount,
        currency: event.currency,
        createdAt: event.occurredAt,
        searchText,
      },
      update: {
        // 冪等性: 同じイベントを再処理しても同じ結果
        items: JSON.stringify(items),
        totalAmount,
        searchText,
      },
    });

    // ユーザー統計も更新
    await this.updateUserStats(event.userId);
  }

  // OrderSubmittedイベント → ステータス更新
  async onOrderSubmitted(event: OrderSubmittedEvent): Promise<void> {
    await this.db.orderReadModel.updateMany({
      where: { orderId: event.orderId },
      data: {
        status: 'pending_payment',
        submittedAt: event.occurredAt,
      },
    });
  }

  // OrderCompletedイベント → 完了処理
  async onOrderCompleted(event: OrderCompletedEvent): Promise<void> {
    await this.db.orderReadModel.updateMany({
      where: { orderId: event.orderId },
      data: {
        status: 'completed',
        completedAt: event.occurredAt,
      },
    });

    // ユーザー統計を更新
    await this.updateUserStats(event.userId);
  }

  // ユーザー統計の再計算
  private async updateUserStats(userId: string): Promise<void> {
    const stats = await this.db.orderReadModel.aggregate({
      where: { userId },
      _count: { orderId: true },
      _sum: { totalAmount: true },
      _avg: { totalAmount: true },
      _max: { createdAt: true },
    });

    const completedCount = await this.db.orderReadModel.count({
      where: { userId, status: 'completed' },
    });

    await this.db.userOrderStats.upsert({
      where: { userId },
      create: {
        userId,
        totalOrders: stats._count.orderId,
        completedOrders: completedCount,
        totalSpent: stats._sum.totalAmount ?? 0,
        averageOrderValue: stats._avg.totalAmount ?? 0,
        lastOrderAt: stats._max.createdAt ?? undefined,
      },
      update: {
        totalOrders: stats._count.orderId,
        completedOrders: completedCount,
        totalSpent: stats._sum.totalAmount ?? 0,
        averageOrderValue: stats._avg.totalAmount ?? 0,
        lastOrderAt: stats._max.createdAt ?? undefined,
      },
    });
  }

  // Projection再構築（全イベントを最初から再生）
  async rebuild(fromEventId?: string): Promise<{ processed: number }> {
    logger.info({ fromEventId }, 'Starting Projection rebuild');

    // 再構築時は既存の読み取りモデルをクリア（または段階的更新）
    if (!fromEventId) {
      await this.db.orderReadModel.deleteMany({});
      await this.db.userOrderStats.deleteMany({});
    }

    let processed = 0;
    let cursor: string | undefined = fromEventId;

    while (true) {
      const events = await this.db.domainEvent.findMany({
        where: {
          eventType: { in: ['OrderCreated', 'OrderSubmitted', 'OrderCompleted'] },
          ...(cursor ? { id: { gt: cursor } } : {}),
        },
        orderBy: { id: 'asc' },
        take: 100,
      });

      if (events.length === 0) break;

      for (const event of events) {
        const payload = JSON.parse(event.payload);
        switch (event.eventType) {
          case 'OrderCreated':
            await this.onOrderCreated(payload as OrderCreatedEvent);
            break;
          case 'OrderSubmitted':
            await this.onOrderSubmitted(payload as OrderSubmittedEvent);
            break;
          case 'OrderCompleted':
            await this.onOrderCompleted(payload as OrderCompletedEvent);
            break;
        }
        processed++;
      }

      cursor = events[events.length - 1].id;
      logger.info({ processed, cursor }, 'Rebuild progress');
    }

    logger.info({ processed }, 'Projection rebuild completed');
    return { processed };
  }
}
```

```typescript
// src/readmodel/orderQueryService.ts — クエリサービス（読み取り専用）

export class OrderQueryService {
  constructor(private readonly db: PrismaClient) {}

  // ユーザーの注文一覧（最適化済みクエリ）
  async getUserOrders(
    userId: string,
    options: { status?: string; cursor?: string; limit?: number } = {}
  ): Promise<PaginatedResult<OrderReadModel>> {
    const limit = Math.min(options.limit ?? 20, 100);

    const rows = await this.db.orderReadModel.findMany({
      where: {
        userId,
        ...(options.status ? { status: options.status } : {}),
        ...(options.cursor ? { createdAt: { lt: new Date(atob(options.cursor)) } } : {}),
      },
      orderBy: { createdAt: 'desc' },
      take: limit + 1,
    });

    const hasMore = rows.length > limit;
    const items = (hasMore ? rows.slice(0, limit) : rows).map(row => ({
      ...row,
      items: JSON.parse(row.items as string),
    }));

    return {
      items: items as OrderReadModel[],
      hasMore,
      nextCursor: hasMore
        ? btoa(items[items.length - 1].createdAt.toISOString())
        : undefined,
    };
  }

  // 全文検索（searchTextインデックス使用）
  async searchOrders(
    query: string,
    options: { limit?: number } = {}
  ): Promise<OrderReadModel[]> {
    const rows = await this.db.orderReadModel.findMany({
      where: {
        searchText: { contains: query.toLowerCase() },
      },
      orderBy: { createdAt: 'desc' },
      take: options.limit ?? 20,
    });

    return rows.map(row => ({ ...row, items: JSON.parse(row.items as string) })) as OrderReadModel[];
  }

  // ユーザー統計（事前計算済み）
  async getUserStats(userId: string): Promise<UserOrderStats | null> {
    return this.db.userOrderStats.findUnique({ where: { userId } });
  }

  // 管理者ダッシュボード用集計
  async getDashboardSummary(): Promise<{
    totalOrders: number;
    pendingOrders: number;
    totalRevenue: number;
    avgOrderValue: number;
  }> {
    const [total, pending, revenue] = await Promise.all([
      this.db.orderReadModel.count(),
      this.db.orderReadModel.count({ where: { status: 'pending_payment' } }),
      this.db.orderReadModel.aggregate({ _sum: { totalAmount: true }, _avg: { totalAmount: true } }),
    ]);

    return {
      totalOrders: total,
      pendingOrders: pending,
      totalRevenue: revenue._sum.totalAmount ?? 0,
      avgOrderValue: revenue._avg.totalAmount ?? 0,
    };
  }
}

// APIエンドポイント（書き込みと読み取りを明確に分離）
router.get('/api/orders', requireAuth, async (req, res) => {
  const result = await orderQueryService.getUserOrders(
    req.user.id,
    { status: req.query.status as string, cursor: req.query.cursor as string }
  );
  res.json(result);
});

// 管理API: Projection再構築
router.post('/api/admin/projections/orders/rebuild', requireAdmin, async (req, res) => {
  const result = await orderProjection.rebuild();
  res.json({ message: 'Rebuild completed', ...result });
});
```

---

## まとめ

Claude Codeで読み取りモデルProjectionを設計する：

1. **CLAUDE.md** に読み取りモデルは書き込みモデルとは独立・ドメインイベント駆動でProjection更新・冪等性必須・再構築可能な設計を明記
2. **非正規化で高速読み取り** ——`userEmail`・`productName`を注文読み取りモデルに埋め込み、一切JOINなしで注文一覧を返せる。Userテーブルが遅くなっても注文一覧に影響しない
3. **`searchText`列** で全文検索を高速化——`orderId + userEmail + productNames`を結合してLOWERCASEで保存、LIKEクエリでインデックスを活用。Elasticsearchなしで検索機能を実装
4. **再構築機能（`rebuild()`）** で読み取りモデルをリセット可能——バグ修正後や新機能追加後にProjectionを全イベントから再生して最新状態に再構築。Event Sourcing+Projectionの最大メリット

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
