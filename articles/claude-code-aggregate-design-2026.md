---
title: "Claude Codeで集約を設計する：集約境界の決定・不変条件の保護・集約間の参照"
emoji: "🏛️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-19 11:00"
---

## はじめに

「集約の境界をどこで引くか迷う」「集約Aから集約Bの内部を変更している」——DDDの集約設計で不変条件を守り、集約間の正しい参照方法を設計をClaude Codeに生成させる。

---

## CLAUDE.mdに集約設計ルールを書く

```markdown
## 集約設計ルール

### 集約境界の決定基準
- 同時に変更される必要があるエンティティをひとまとめにする
- 集約は小さく保つ（大きすぎると競合が多発）
- 集約のルート（Aggregate Root）のみが外部から直接参照される

### 不変条件
- 集約内の全エンティティの整合性を保証するビジネスルール
- 不変条件の検証は集約ルートのメソッドで行う
- 外部から集約の内部エンティティを直接変更しない

### 集約間の参照
- 集約間はIDのみで参照（直接の集約オブジェクト参照禁止）
- 複数集約にまたがる操作はアプリケーション層（ユースケース）で調整
- 集約間の整合性は結果整合（eventual consistency）
```

---

## 集約設計実装の生成

```
集約設計のベストプラクティスを実装してください。

要件：
- 集約ルートによる不変条件の保護
- 内部エンティティへのアクセス制限
- 集約間のID参照
- コレクション変更の制御

生成ファイル: src/domain/aggregates/
```

---

## 生成される集約設計実装

```typescript
// src/domain/aggregates/order/order.ts — 集約ルート

// エラー: OrderItemは集約内部エンティティ（外部から直接操作しない）
class OrderItem {
  private constructor(
    private readonly _id: string,
    private readonly _productId: string,    // 商品集約のID参照（集約オブジェクトは持たない）
    private _quantity: number,
    private readonly _price: Money
  ) {}

  static create(productId: string, quantity: number, price: Money): OrderItem {
    if (quantity <= 0) throw new DomainError('Quantity must be positive');
    return new OrderItem(ulid(), productId, quantity, price);
  }

  increaseQuantity(amount: number): void {
    if (amount <= 0) throw new DomainError('Increase amount must be positive');
    this._quantity += amount;
  }

  get id(): string { return this._id; }
  get productId(): string { return this._productId; }
  get quantity(): number { return this._quantity; }
  get subtotal(): Money { return this._price.multiply(this._quantity); }
}

// 集約ルート
export class Order extends AggregateRoot {
  private _items: OrderItem[];
  private _status: OrderStatus;

  private constructor(
    private readonly _id: string,
    private readonly _userId: string,    // User集約のID参照
    items: OrderItem[],
    status: OrderStatus
  ) {
    super();
    this._items = items;
    this._status = status;
  }

  static create(userId: string, initialItems: Array<{ productId: string; quantity: number; price: Money }>): Order {
    if (initialItems.length === 0) {
      throw new DomainError('Order must have at least one item');
    }
    if (initialItems.length > 100) {
      throw new DomainError('Order cannot have more than 100 items');
    }

    const items = initialItems.map(i => OrderItem.create(i.productId, i.quantity, i.price));
    const order = new Order(ulid(), userId, items, 'draft');

    order.addDomainEvent({
      eventId: ulid(),
      eventType: 'OrderCreated',
      occurredAt: new Date(),
      orderId: order._id,
      userId,
      totalAmount: order.total,
    });

    return order;
  }

  // 集約ルートを通じた操作（内部エンティティを外部から直接変更しない）
  addItem(productId: string, quantity: number, price: Money): void {
    if (this._status !== 'draft') {
      throw new DomainError(`Cannot add items to order in status '${this._status}'`);
    }

    // 不変条件: アイテム数の上限
    if (this._items.length >= 100) {
      throw new DomainError('Order cannot have more than 100 items');
    }

    const existingItem = this._items.find(i => i.productId === productId);
    if (existingItem) {
      // 既存商品の数量を増やす
      existingItem.increaseQuantity(quantity);
    } else {
      this._items.push(OrderItem.create(productId, quantity, price));
    }
  }

  removeItem(productId: string): void {
    if (this._status !== 'draft') {
      throw new DomainError(`Cannot remove items from order in status '${this._status}'`);
    }

    const index = this._items.findIndex(i => i.productId === productId);
    if (index === -1) throw new NotFoundError(`Item with productId ${productId} not found`);

    this._items.splice(index, 1);

    // 不変条件: アイテムが0になったらキャンセル
    if (this._items.length === 0) {
      this._status = 'cancelled';
    }
  }

  submit(): void {
    if (this._status !== 'draft') throw new DomainError('Can only submit draft orders');

    // 不変条件: 提出前に少なくとも1アイテムが必要
    if (this._items.length === 0) throw new DomainError('Cannot submit empty order');

    this._status = 'pending_payment';
    this.addDomainEvent({
      eventId: ulid(), eventType: 'OrderSubmitted', occurredAt: new Date(),
      orderId: this._id, total: this.total,
    });
  }

  // コレクションを外部に返す場合は読み取り専用コピー（防御的コピー）
  get items(): ReadonlyArray<{ id: string; productId: string; quantity: number; subtotal: Money }> {
    return this._items.map(item => ({
      id: item.id,
      productId: item.productId,
      quantity: item.quantity,
      subtotal: item.subtotal,
    }));
  }

  get total(): Money {
    if (this._items.length === 0) return Money.zero('JPY');
    return this._items.reduce((sum, item) => sum.add(item.subtotal), Money.zero('JPY'));
  }

  get id(): string { return this._id; }
  get userId(): string { return this._userId; }
  get status(): OrderStatus { return this._status; }
}

// 間違った設計の例（集約外から内部を操作 — やってはいけない）
// ❌ order.items[0].increaseQuantity(2); // 直接OrderItemを変更

// 正しい設計
// ✅ order.addItem('prod-123', 2, price); // 集約ルート経由で操作
```

```typescript
// 集約間の正しい参照（IDのみ）

// ❌ 間違い: 集約Aが集約Bのオブジェクトを直接持つ
class Order {
  private _user: User;  // Userオブジェクトを直接参照 → やってはいけない
}

// ✅ 正しい: IDのみを持ち、必要な時にリポジトリで取得
class Order {
  private readonly _userId: string;  // UserのIDのみ
}

// 複数集約にまたがる操作はアプリケーション層で調整
export class ShipOrderUseCase {
  async execute(orderId: string): Promise<void> {
    return UnitOfWorkFactory.run(async (uow) => {
      // Order集約
      const order = await uow.orderRepository.findById(orderId);

      // Inventory集約（Order集約は直接持たない、IDで取得）
      for (const item of order.items) {
        const inventory = await uow.inventoryRepository.findByProductId(item.productId);
        inventory.deductStock(Quantity.of(item.quantity, 'piece'));
        await uow.inventoryRepository.save(inventory);
      }

      order.ship();
      await uow.orderRepository.save(order);
    });
  }
}
```

---

## まとめ

Claude Codeで集約を設計する：

1. **CLAUDE.md** に集約ルートのみが外部から参照可能・内部エンティティへの直接変更禁止・集約間はIDのみで参照・集約間の操作はアプリ層で調整を明記
2. **不変条件をメソッドで保護** `order.addItem()`内でアイテム数100未満チェック——外部から`order._items.push()`を許してしまうと不変条件を守れない。状態変更は必ず集約ルートのメソッド経由
3. **防御的コピー（get items）** で内部コレクションをReadonlyArrayとして公開——外部が`order.items.push(newItem)`でコレクションを直接変更できないようにする。内部状態を完全にカプセル化
4. **集約間のIDのみ参照** でトランザクション境界を明確化——`order._userId`はUserオブジェクトではなくstring。UserのロードはリポジトリがUser集約のトランザクションで行い、Orderのトランザクションに混在しない

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
