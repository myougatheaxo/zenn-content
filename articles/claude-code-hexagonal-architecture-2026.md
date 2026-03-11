---
title: "Claude Codeでヘキサゴナルアーキテクチャを設計する：ポート&アダプター・依存関係の逆転・テスト可能性"
emoji: "⬡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-20 15:00"
---

## はじめに

「ビジネスロジックがExpressやPrismaに直接依存している」「テストのためにDBを起動しなければならない」——ヘキサゴナルアーキテクチャ（ポート&アダプター）でビジネスロジックをインフラから完全に切り離す設計をClaude Codeに生成させる。

---

## CLAUDE.mdにヘキサゴナルアーキテクチャ設計ルールを書く

```markdown
## ヘキサゴナルアーキテクチャ設計ルール

### レイヤー構成
- Domain（最内層）: ビジネスロジック、集約、値オブジェクト、ドメインサービス
- Application（中間層）: ユースケース（入力ポート）、出力ポートインターフェース
- Infrastructure（最外層）: アダプター（DB、HTTP、メッセージキュー等）

### ポートの種類
- 入力ポート（Driving Port）: アプリケーションへの入口（UseCase interface）
- 出力ポート（Driven Port）: アプリケーションが依存する外部インターフェース（Repository interface）

### 依存ルール
- 内側のレイヤーは外側を知らない
- インフラはアプリケーション/ドメインに依存（逆は禁止）
- テスト時はインフラをスタブに差し替え可能
```

---

## ヘキサゴナルアーキテクチャ実装の生成

```
ヘキサゴナルアーキテクチャを設計してください。

要件：
- ポート&アダプターパターン
- 依存逆転の原則
- 複数のドライビングアダプター（HTTP / CLI / テスト）
- 複数のドリブンアダプター（Prisma / InMemory）

生成ファイル: src/domain/ + src/application/ + src/infrastructure/ + src/adapters/
```

---

## 生成されるヘキサゴナルアーキテクチャ実装

```typescript
// ===== Domain Layer =====
// src/domain/order/order.ts

export class Order {
  private constructor(
    readonly id: string,
    private _status: OrderStatus,
    private _items: OrderItem[],
    readonly userId: string,
    readonly createdAt: Date
  ) {}

  static create(userId: string, items: OrderItemInput[]): Order {
    if (items.length === 0) throw new DomainError('Order must have at least one item');
    const orderItems = items.map(i => OrderItem.create(i.productId, i.quantity, i.price));
    return new Order(ulid(), 'draft', orderItems, userId, new Date());
  }

  submit(): void {
    if (this._status !== 'draft') throw new DomainError('Only draft orders can be submitted');
    this._status = 'pending_payment';
  }

  get status() { return this._status; }
  get items(): ReadonlyArray<OrderItem> { return this._items; }
  get total(): Money {
    return this._items.reduce((sum, i) => sum.add(i.subtotal), Money.zero('JPY'));
  }
}
```

```typescript
// ===== Application Layer (Ports) =====
// src/application/ports/input/placeOrderUseCase.ts — 入力ポート

export interface PlaceOrderInput {
  userId: string;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
  webhookUrl?: string;
}

export interface PlaceOrderOutput {
  orderId: string;
  status: string;
  total: number;
}

// ユースケースインターフェース（入力ポート）
export interface IPlaceOrderUseCase {
  execute(input: PlaceOrderInput): Promise<PlaceOrderOutput>;
}
```

```typescript
// src/application/ports/output/orderRepository.ts — 出力ポート

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
  findByUserId(userId: string): Promise<Order[]>;
}

// src/application/ports/output/paymentGateway.ts — 出力ポート
export interface IPaymentGateway {
  createPaymentIntent(amount: Money, userId: string): Promise<string>;
}

// src/application/ports/output/eventPublisher.ts — 出力ポート
export interface IEventPublisher {
  publish(event: DomainEvent): Promise<void>;
}
```

```typescript
// src/application/usecases/placeOrderUseCase.ts — ユースケース実装（アプリケーション層）
// ※ ドメイン層とポートのみに依存（ExpressもPrismaも知らない）

export class PlaceOrderUseCase implements IPlaceOrderUseCase {
  constructor(
    // 出力ポート（インターフェース）を注入——具体実装は外側が決める
    private readonly orderRepository: IOrderRepository,
    private readonly paymentGateway: IPaymentGateway,
    private readonly eventPublisher: IEventPublisher
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderOutput> {
    // 1. ドメインオブジェクト生成
    const order = Order.create(
      input.userId,
      input.items.map(i => ({
        productId: i.productId,
        quantity: i.quantity,
        price: Money.of(i.unitPrice, 'JPY'),
      }))
    );

    // 2. 支払い意図を作成
    const paymentIntentId = await this.paymentGateway.createPaymentIntent(
      order.total,
      input.userId
    );

    // 3. 注文を確定
    order.submit();

    // 4. 保存
    await this.orderRepository.save(order);

    // 5. イベント発行
    await this.eventPublisher.publish({
      eventType: 'OrderPlaced',
      orderId: order.id,
      userId: input.userId,
      total: order.total.amount,
      paymentIntentId,
    });

    return {
      orderId: order.id,
      status: order.status,
      total: order.total.amount,
    };
  }
}
```

```typescript
// ===== Infrastructure Layer (Adapters) =====
// src/adapters/driving/http/orderController.ts — ドライビングアダプター（HTTP）
// アプリケーション層の入力ポートを呼び出すだけ

export class OrderController {
  constructor(private readonly placeOrder: IPlaceOrderUseCase) {}

  async postOrder(req: Request, res: Response): Promise<void> {
    const result = await this.placeOrder.execute({
      userId: req.user.id,
      items: req.body.items,
      webhookUrl: req.body.webhookUrl,
    });
    res.status(201).json(result);
  }
}

// src/adapters/driven/db/prismaOrderRepository.ts — ドリブンアダプター（DB）
export class PrismaOrderRepository implements IOrderRepository {
  constructor(private readonly db: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const row = await this.db.order.findUnique({ where: { id }, include: { items: true } });
    return row ? OrderMapper.toDomain(row) : null;
  }

  async save(order: Order): Promise<void> {
    await this.db.order.upsert({
      where: { id: order.id },
      create: OrderMapper.toCreateData(order),
      update: OrderMapper.toUpdateData(order),
    });
  }

  async findByUserId(userId: string): Promise<Order[]> {
    const rows = await this.db.order.findMany({ where: { userId }, include: { items: true } });
    return rows.map(OrderMapper.toDomain);
  }
}

// ===== Dependency Injection (Composition Root) =====
// src/main.ts — 依存の組み立て（コンポジションルート）
// ここだけがインフラを知る

const prismaDb = new PrismaClient();
const stripeGateway = new StripePaymentGateway(process.env.STRIPE_KEY!);
const eventBus = new RedisEventPublisher(redis);

const orderRepo = new PrismaOrderRepository(prismaDb);

const placeOrderUseCase = new PlaceOrderUseCase(
  orderRepo,
  stripeGateway,
  eventBus
);

const orderController = new OrderController(placeOrderUseCase);
app.post('/api/orders', (req, res) => orderController.postOrder(req, res));

// ===== テスト用のスタブアダプター =====
// src/adapters/driven/testing/stubOrderRepository.ts
class StubOrderRepository implements IOrderRepository {
  private readonly store = new Map<string, Order>();
  async findById(id: string) { return this.store.get(id) ?? null; }
  async save(order: Order) { this.store.set(order.id, order); }
  async findByUserId(userId: string) {
    return [...this.store.values()].filter(o => o.userId === userId);
  }
}

class StubPaymentGateway implements IPaymentGateway {
  async createPaymentIntent(amount: Money, userId: string): Promise<string> {
    return `stub-pi-${ulid()}`;
  }
}

class SpyEventPublisher implements IEventPublisher {
  readonly published: DomainEvent[] = [];
  async publish(event: DomainEvent): Promise<void> { this.published.push(event); }
}

// テスト: DBもStripeも不要
describe('PlaceOrderUseCase', () => {
  it('注文が正常に作成される', async () => {
    const orderRepo = new StubOrderRepository();
    const eventPublisher = new SpyEventPublisher();
    const useCase = new PlaceOrderUseCase(
      orderRepo,
      new StubPaymentGateway(),
      eventPublisher
    );

    const result = await useCase.execute({
      userId: 'user-1',
      items: [{ productId: 'prod-1', quantity: 2, unitPrice: 1000 }],
    });

    expect(result.status).toBe('pending_payment');
    expect(result.total).toBe(2000);
    expect(eventPublisher.published).toHaveLength(1);
    expect(eventPublisher.published[0].eventType).toBe('OrderPlaced');
  });
});
```

---

## まとめ

Claude Codeでヘキサゴナルアーキテクチャを設計する：

1. **CLAUDE.md** にドメイン→アプリケーション→インフラの一方向依存・入力ポート（UseCase interface）と出力ポート（Repository/Gateway interface）の区別・インフラはポートを実装するアダプターを明記
2. **PlaceOrderUseCaseはPrismaもExpressも知らない** ——`IOrderRepository`・`IPaymentGateway`のインターフェースだけに依存。決済プロバイダーをStripeからPayPayに変えてもUseCaseのコードは変わらない
3. **コンポジションルート（main.ts）** がインフラを唯一知る場所——`new PrismaOrderRepository()`・`new StripePaymentGateway()`はmain.tsでのみ呼ばれ、UseCaseはインターフェースのみを受け取る
4. **スタブアダプター** でDBなしの超高速テスト——`StubOrderRepository`・`SpyEventPublisher`をDI注入するだけで完全なユースケーステストが完結。CI環境でDBコンテナが不要で10倍以上高速

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
