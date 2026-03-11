---
title: "Claude Codeでリポジトリパターンを設計する：クエリメソッド・仕様パターン統合・テスト可能なリポジトリ"
emoji: "🗄️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-20 09:00"
---

## はじめに

「データアクセスロジックがサービス層に散在している」「テストのたびにDBをモックするのが面倒」——リポジトリパターンでデータアクセスを抽象化し、ドメイン層をインフラから切り離す設計をClaude Codeに生成させる。

---

## CLAUDE.mdにリポジトリパターン設計ルールを書く

```markdown
## リポジトリパターン設計ルール

### インターフェース設計
- findById, findOne, findMany, save, delete の基本5メソッド
- 複雑なクエリは仕様パターン（Specification）と組み合わせる
- ページネーション: cursor方式（offset禁止）

### 実装の配置
- IOrderRepository: domain層（インターフェースのみ）
- PrismaOrderRepository: infrastructure層（実装）
- InMemoryOrderRepository: テスト用（infrastructure/testing/）

### 命名規則
- findById(id): 単一取得（null許容）
- findByIdOrThrow(id): 単一取得（NotFoundErrorをthrow）
- findMany(spec?, pagination?): 複数取得
- save(entity): 新規作成・更新（upsert）
- delete(id): 削除
```

---

## リポジトリ実装の生成

```
リポジトリパターンを設計してください。

要件：
- 型安全なクエリインターフェース
- 仕様パターン統合
- カーソルページネーション
- インメモリテスト実装

生成ファイル: src/domain/repositories/ + src/infrastructure/repositories/
```

---

## 生成されるリポジトリ実装

```typescript
// src/domain/repositories/orderRepository.ts — ドメインインターフェース

export interface OrderCriteria {
  userId?: string;
  status?: OrderStatus | OrderStatus[];
  totalMin?: Money;
  totalMax?: Money;
  createdAfter?: Date;
  createdBefore?: Date;
}

export interface OrderSort {
  field: 'createdAt' | 'total' | 'status';
  direction: 'asc' | 'desc';
}

export interface PaginationOptions {
  cursor?: string;   // Base64エンコードされたカーソル
  limit: number;
}

export interface PaginatedResult<T> {
  items: T[];
  nextCursor?: string;
  hasMore: boolean;
  total?: number;
}

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  findByIdOrThrow(id: string): Promise<Order>;
  findOne(criteria: OrderCriteria): Promise<Order | null>;
  findMany(
    criteria?: OrderCriteria,
    sort?: OrderSort,
    pagination?: PaginationOptions
  ): Promise<PaginatedResult<Order>>;
  count(criteria?: OrderCriteria): Promise<number>;
  save(order: Order): Promise<void>;
  delete(id: string): Promise<void>;
  existsById(id: string): Promise<boolean>;
}
```

```typescript
// src/infrastructure/repositories/prismaOrderRepository.ts — Prisma実装

export class PrismaOrderRepository implements IOrderRepository {
  constructor(private readonly db: PrismaClient | PrismaTransactionClient) {}

  async findById(id: string): Promise<Order | null> {
    const row = await this.db.order.findUnique({
      where: { id },
      include: { items: true },
    });
    return row ? OrderMapper.toDomain(row) : null;
  }

  async findByIdOrThrow(id: string): Promise<Order> {
    const order = await this.findById(id);
    if (!order) throw new NotFoundError('Order', id);
    return order;
  }

  async findOne(criteria: OrderCriteria): Promise<Order | null> {
    const row = await this.db.order.findFirst({
      where: this.buildWhere(criteria),
      include: { items: true },
    });
    return row ? OrderMapper.toDomain(row) : null;
  }

  async findMany(
    criteria: OrderCriteria = {},
    sort: OrderSort = { field: 'createdAt', direction: 'desc' },
    pagination: PaginationOptions = { limit: 20 }
  ): Promise<PaginatedResult<Order>> {
    const limit = Math.min(pagination.limit, 100);
    const cursor = pagination.cursor ? this.decodeCursor(pagination.cursor) : undefined;

    const rows = await this.db.order.findMany({
      where: {
        ...this.buildWhere(criteria),
        // カーソルページネーション
        ...(cursor ? this.buildCursorWhere(cursor, sort) : {}),
      },
      include: { items: true },
      orderBy: this.buildOrderBy(sort),
      take: limit + 1,  // hasMore判定のため1件余分に取得
    });

    const hasMore = rows.length > limit;
    const items = hasMore ? rows.slice(0, limit) : rows;
    const nextCursor = hasMore ? this.encodeCursor(items[items.length - 1], sort) : undefined;

    return {
      items: items.map(OrderMapper.toDomain),
      nextCursor,
      hasMore,
    };
  }

  async count(criteria: OrderCriteria = {}): Promise<number> {
    return this.db.order.count({ where: this.buildWhere(criteria) });
  }

  async save(order: Order): Promise<void> {
    await this.db.order.upsert({
      where: { id: order.id },
      create: OrderMapper.toCreateData(order),
      update: OrderMapper.toUpdateData(order),
    });

    // OrderItemsの同期
    await this.syncItems(order);
  }

  async delete(id: string): Promise<void> {
    await this.db.order.delete({ where: { id } });
  }

  async existsById(id: string): Promise<boolean> {
    const count = await this.db.order.count({ where: { id } });
    return count > 0;
  }

  // クエリビルダー
  private buildWhere(criteria: OrderCriteria): Prisma.OrderWhereInput {
    return {
      ...(criteria.userId ? { userId: criteria.userId } : {}),
      ...(criteria.status
        ? Array.isArray(criteria.status)
          ? { status: { in: criteria.status } }
          : { status: criteria.status }
        : {}),
      ...(criteria.totalMin || criteria.totalMax
        ? {
            total: {
              ...(criteria.totalMin ? { gte: criteria.totalMin.amount } : {}),
              ...(criteria.totalMax ? { lte: criteria.totalMax.amount } : {}),
            },
          }
        : {}),
      ...(criteria.createdAfter ? { createdAt: { gte: criteria.createdAfter } } : {}),
      ...(criteria.createdBefore ? { createdAt: { lte: criteria.createdBefore } } : {}),
    };
  }

  private buildOrderBy(sort: OrderSort): Prisma.OrderOrderByWithRelationInput {
    return { [sort.field]: sort.direction };
  }

  private buildCursorWhere(
    cursor: { createdAt: Date; id: string },
    sort: OrderSort
  ): Prisma.OrderWhereInput {
    // 複合カーソル: (createdAt DESC, id DESC) の場合
    if (sort.field === 'createdAt' && sort.direction === 'desc') {
      return {
        OR: [
          { createdAt: { lt: cursor.createdAt } },
          { createdAt: cursor.createdAt, id: { lt: cursor.id } },
        ],
      };
    }
    // 他のソートも同様に実装
    return {};
  }

  private encodeCursor(row: { createdAt: Date; id: string }, sort: OrderSort): string {
    const payload = { createdAt: row.createdAt.toISOString(), id: row.id, sort };
    return Buffer.from(JSON.stringify(payload)).toString('base64url');
  }

  private decodeCursor(cursor: string): { createdAt: Date; id: string } {
    const payload = JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8'));
    return { createdAt: new Date(payload.createdAt), id: payload.id };
  }

  private async syncItems(order: Order): Promise<void> {
    // 既存アイテムを削除して再挿入（シンプルな同期戦略）
    await this.db.orderItem.deleteMany({ where: { orderId: order.id } });
    if (order.items.length > 0) {
      await this.db.orderItem.createMany({
        data: order.items.map(item => OrderMapper.toItemCreateData(order.id, item)),
      });
    }
  }
}
```

```typescript
// src/infrastructure/repositories/inMemoryOrderRepository.ts — テスト用

export class InMemoryOrderRepository implements IOrderRepository {
  private readonly store = new Map<string, Order>();

  async findById(id: string): Promise<Order | null> {
    return this.store.get(id) ?? null;
  }

  async findByIdOrThrow(id: string): Promise<Order> {
    const order = this.store.get(id);
    if (!order) throw new NotFoundError('Order', id);
    return order;
  }

  async findOne(criteria: OrderCriteria): Promise<Order | null> {
    const results = await this.findMany(criteria, undefined, { limit: 1 });
    return results.items[0] ?? null;
  }

  async findMany(
    criteria: OrderCriteria = {},
    sort: OrderSort = { field: 'createdAt', direction: 'desc' },
    pagination: PaginationOptions = { limit: 20 }
  ): Promise<PaginatedResult<Order>> {
    let items = [...this.store.values()];

    // フィルタリング
    if (criteria.userId) items = items.filter(o => o.userId === criteria.userId);
    if (criteria.status) {
      const statuses = Array.isArray(criteria.status) ? criteria.status : [criteria.status];
      items = items.filter(o => statuses.includes(o.status));
    }

    // ソート
    items.sort((a, b) => {
      const dir = sort.direction === 'asc' ? 1 : -1;
      if (sort.field === 'createdAt') {
        return dir * (a.createdAt.getTime() - b.createdAt.getTime());
      }
      return 0;
    });

    // ページネーション（シンプルなindex版、テスト用）
    const start = pagination.cursor ? parseInt(atob(pagination.cursor)) : 0;
    const slice = items.slice(start, start + pagination.limit + 1);
    const hasMore = slice.length > pagination.limit;
    const pageItems = hasMore ? slice.slice(0, pagination.limit) : slice;
    const nextCursor = hasMore ? btoa(String(start + pagination.limit)) : undefined;

    return { items: pageItems, hasMore, nextCursor };
  }

  async count(criteria: OrderCriteria = {}): Promise<number> {
    const results = await this.findMany(criteria, undefined, { limit: 10000 });
    return results.items.length;
  }

  async save(order: Order): Promise<void> {
    this.store.set(order.id, order);
  }

  async delete(id: string): Promise<void> {
    this.store.delete(id);
  }

  async existsById(id: string): Promise<boolean> {
    return this.store.has(id);
  }

  // テストヘルパー
  clear(): void { this.store.clear(); }
  size(): number { return this.store.size; }
  all(): Order[] { return [...this.store.values()]; }
}

// テストでの使用例
describe('OrderService', () => {
  let orderRepo: InMemoryOrderRepository;
  let service: OrderService;

  beforeEach(() => {
    orderRepo = new InMemoryOrderRepository();
    service = new OrderService(orderRepo);  // DBなしでテスト可能
  });

  it('ユーザーの注文一覧を取得できる', async () => {
    await orderRepo.save(buildOrder({ userId: 'user-1', status: 'pending_payment' }));
    await orderRepo.save(buildOrder({ userId: 'user-1', status: 'completed' }));
    await orderRepo.save(buildOrder({ userId: 'user-2', status: 'pending_payment' }));

    const result = await service.getUserOrders('user-1');
    expect(result.items).toHaveLength(2);
  });
});
```

---

## まとめ

Claude Codeでリポジトリパターンを設計する：

1. **CLAUDE.md** にIOrderRepositoryをdomain層・PrismaOrderRepositoryをinfrastructure層・InMemoryOrderRepositoryをテスト用で配置・offset禁止・cursor方式を明記
2. **`findByIdOrThrow()`** で404ハンドリングを集約——呼び出し側が`if (!order)`を書く必要なく、NotFoundErrorが自動で飛ぶ。サービス層のコードが簡潔になる
3. **`buildWhere()`クエリビルダー** で型安全なフィルタリング——`criteria.status`が`string | string[]`どちらでも正しいPrismaクエリを生成。nullチェックもインターフェースで強制
4. **`InMemoryOrderRepository`** でDBなしの高速テスト——`new InMemoryOrderRepository()`をDI注入するだけでサービス層テストが完結。DB接続不要で10倍速い

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
