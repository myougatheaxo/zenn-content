---
title: "Claude CodeでUnit of Workパターンを設計する：複数リポジトリのトランザクション統一・変更追跡・原子コミット"
emoji: "🔒"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 22:00"
---

## はじめに

「複数のリポジトリを使う処理でトランザクションが統一されていなかった」——Unit of Workパターンで複数リポジトリへの変更を1つのトランザクションにまとめ、原子的にコミット/ロールバックする設計をClaude Codeに生成させる。

---

## CLAUDE.mdにUnit of Work設計ルールを書く

```markdown
## Unit of Workパターン設計ルール

### 役割
- 1つのユースケース（トランザクション）スコープを管理
- スコープ内で取得した集約を追跡（変更検出）
- commit()で全変更を一括保存、rollback()で全変更を取り消し

### リポジトリとの関係
- リポジトリはUoWが管理するDBコネクション/Prismaトランザクションを使用
- UoWを通じてリポジトリを取得する（UoW.getOrderRepository()）
- UoW外でリポジトリを直接使用しない

### Prismaとの統合
- Prisma.$transaction(async (tx) => { ... }) をUoWのスコープとして使用
- リポジトリはPrismaトランザクション（tx）を受け取る
```

---

## Unit of Work実装の生成

```
Unit of Workパターンを設計してください。

要件：
- Prismaトランザクションとの統合
- リポジトリアクセスのファクトリー
- ドメインイベントの収集とコミット後の発行
- エラー時の自動ロールバック

生成ファイル: src/infrastructure/unitOfWork/
```

---

## 生成されるUnit of Work実装

```typescript
// src/infrastructure/unitOfWork/unitOfWork.ts — Unit of Work

export interface IUnitOfWork {
  orderRepository: IOrderRepository;
  userRepository: IUserRepository;
  inventoryRepository: IInventoryRepository;
  commit(): Promise<void>;
  rollback(): Promise<void>;
}

export class PrismaUnitOfWork implements IUnitOfWork {
  private _orderRepository: IOrderRepository | null = null;
  private _userRepository: IUserRepository | null = null;
  private _inventoryRepository: IInventoryRepository | null = null;
  private _aggregates: AggregateRoot[] = [];
  private _committed = false;

  constructor(private readonly tx: PrismaTransactionClient) {}

  get orderRepository(): IOrderRepository {
    this._orderRepository ??= new PrismaOrderRepository(this.tx, this);
    return this._orderRepository;
  }

  get userRepository(): IUserRepository {
    this._userRepository ??= new PrismaUserRepository(this.tx);
    return this._userRepository;
  }

  get inventoryRepository(): IInventoryRepository {
    this._inventoryRepository ??= new PrismaInventoryRepository(this.tx);
    return this._inventoryRepository;
  }

  // リポジトリが集約を登録（変更追跡用）
  registerAggregate(aggregate: AggregateRoot): void {
    this._aggregates.push(aggregate);
  }

  async commit(): Promise<void> {
    if (this._committed) throw new Error('Unit of Work already committed');
    this._committed = true;
    // コミット自体はPrismaトランザクションが行う（tx.commit相当）
    // ドメインイベントを収集
    const events = this._aggregates.flatMap(a => a.collectDomainEvents());

    // コミット後のイベント発行はスコープ外で行う
    this._pendingEvents = events;
  }

  private _pendingEvents: DomainEvent[] = [];

  getPendingEvents(): DomainEvent[] {
    return this._pendingEvents;
  }

  async rollback(): Promise<void> {
    // Prismaトランザクションのロールバックはthrowで発動
    this._aggregates = [];
    this._pendingEvents = [];
  }
}

// UoWファクトリー
export class UnitOfWorkFactory {
  static async run<T>(
    fn: (uow: PrismaUnitOfWork) => Promise<T>,
    afterCommit?: (events: DomainEvent[]) => Promise<void>
  ): Promise<T> {
    let pendingEvents: DomainEvent[] = [];

    const result = await prisma.$transaction(async (tx) => {
      const uow = new PrismaUnitOfWork(tx as any);
      const result = await fn(uow);
      await uow.commit();
      pendingEvents = uow.getPendingEvents();
      return result;
    });

    // トランザクション成功後にドメインイベントを発行
    if (pendingEvents.length > 0 && afterCommit) {
      await afterCommit(pendingEvents).catch(err =>
        logger.error({ err }, 'Post-commit event dispatch failed')
      );
    }

    return result;
  }
}
```

```typescript
// src/infrastructure/repositories/orderRepository.ts — UoW対応リポジトリ

export class PrismaOrderRepository implements IOrderRepository {
  constructor(
    private readonly tx: PrismaTransactionClient,
    private readonly uow: PrismaUnitOfWork
  ) {}

  async findById(id: string): Promise<Order | null> {
    const row = await this.tx.order.findUnique({
      where: { id },
      include: { items: true },
    });

    if (!row) return null;

    const order = OrderMapper.toDomain(row);
    this.uow.registerAggregate(order); // 変更追跡に登録
    return order;
  }

  async save(order: Order): Promise<void> {
    await this.tx.order.upsert({
      where: { id: order.id },
      create: OrderMapper.toCreateData(order),
      update: OrderMapper.toUpdateData(order),
    });
  }
}

// アプリケーション層: UoWを使ったユースケース
export class PlaceOrderUseCase {
  async execute(input: PlaceOrderInput): Promise<Order> {
    return UnitOfWorkFactory.run(
      async (uow) => {
        // 同一トランザクション内で複数リポジトリを操作
        const user = await uow.userRepository.findById(input.userId);
        if (!user) throw new NotFoundError('User not found');

        // 在庫確認
        for (const item of input.items) {
          const inventory = await uow.inventoryRepository.findByProductId(item.productId);
          if (!inventory.hasEnough(Quantity.of(item.quantity, 'piece'))) {
            throw new OutOfStockError(item.productId);
          }
        }

        // 注文作成
        const order = Order.create(input.userId, input.items);

        // 在庫を減らす
        for (const item of input.items) {
          const inventory = await uow.inventoryRepository.findByProductId(item.productId);
          inventory.decrement(Quantity.of(item.quantity, 'piece'));
          await uow.inventoryRepository.save(inventory);
        }

        // 注文保存
        await uow.orderRepository.save(order);

        return order;
        // commit()は自動（UnitOfWorkFactory.runが呼ぶ）
      },
      // コミット後にドメインイベントを発行
      async (events) => {
        await eventDispatcher.dispatch(events);
      }
    );
  }
}
```

---

## まとめ

Claude CodeでUnit of Workパターンを設計する：

1. **CLAUDE.md** にUoWがPrismaトランザクションをスコープとして管理・リポジトリはUoWから取得・コミット後にドメインイベントを発行・エラーはthrowでロールバックを明記
2. **`UnitOfWorkFactory.run(tx => { ... })`** でユースケース全体を1トランザクションに——注文作成・在庫更新・ユーザーポイント更新を全て同一`tx`で実行してアトミックに成功/失敗
3. **`registerAggregate`で変更追跡** し、commit()時にドメインイベントを収集——「どの集約が変更されたか」をUoWが把握してイベント収集を自動化
4. **コミット後のイベント発行** で「DBは成功したがイベントが出なかった」問題を防止——トランザクション外でのイベント発行でロールバック済みイベントが流れることもない

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
