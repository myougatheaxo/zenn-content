---
title: "Claude Codeでテストデータビルダーを設計する：Builderパターン・フィクスチャ工場・テストの可読性向上"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "testing", "architecture"]
published: true
published_at: "2026-03-21 13:00"
---

## はじめに

「テストのセットアップコードが長くて何をテストしているか分からない」「必須フィールドを全部埋めないとオブジェクトが作れない」——Builderパターンでテストデータを宣言的に組み立て、テストの本質だけを際立たせる設計をClaude Codeに生成させる。

---

## CLAUDE.mdにテストデータビルダー設計ルールを書く

```markdown
## テストデータビルダー設計ルール

### Builderパターンの原則
- デフォルト値を内蔵（最小限のコードでオブジェクトを作れる）
- チェーンメソッドで必要な部分だけ上書き
- build()で最終オブジェクトを生成

### 命名規則
- ビルダークラス: OrderBuilder, UserBuilder
- ファクトリ関数: buildOrder(), buildUser()（ビルダーのショートカット）
- 状態付きビルダー: buildCompletedOrder(), buildPremiumUser()（よく使うプリセット）

### 配置
- src/__tests__/builders/ に集約
- プロダクションコードに混在させない
- FactoryBotパターン: faker.jsと組み合わせてランダムだが有効なデータ生成
```

---

## テストデータビルダー実装の生成

```
テストデータビルダーを設計してください。

要件：
- Builderパターン（メソッドチェーン）
- デフォルト値と部分上書き
- 関連エンティティの自動構築
- faker.js統合

生成ファイル: src/__tests__/builders/
```

---

## 生成されるテストデータビルダー実装

```typescript
// src/__tests__/builders/orderBuilder.ts — 注文ビルダー

import { faker } from '@faker-js/faker/locale/ja';

// デフォルト値を持つビルダー（必要な部分だけ上書き）
export class OrderBuilder {
  private orderId = ulid();
  private userId = ulid();
  private status: OrderStatus = 'draft';
  private items: Array<{ productId: string; quantity: number; unitPrice: number }> = [
    { productId: ulid(), quantity: 1, unitPrice: 1000 },
  ];
  private createdAt = new Date();
  private completedAt: Date | undefined = undefined;
  private webhookUrl: string | undefined = undefined;

  withId(id: string): this {
    this.orderId = id;
    return this;
  }

  withUser(userId: string): this {
    this.userId = userId;
    return this;
  }

  withStatus(status: OrderStatus): this {
    this.status = status;
    return this;
  }

  withItems(items: typeof this.items): this {
    this.items = items;
    return this;
  }

  withItem(item: { productId?: string; quantity?: number; unitPrice?: number }): this {
    this.items = [{
      productId: item.productId ?? ulid(),
      quantity: item.quantity ?? 1,
      unitPrice: item.unitPrice ?? 1000,
    }];
    return this;
  }

  addItem(item: { productId?: string; quantity?: number; unitPrice?: number }): this {
    this.items.push({
      productId: item.productId ?? ulid(),
      quantity: item.quantity ?? 1,
      unitPrice: item.unitPrice ?? 1000,
    });
    return this;
  }

  withCompletedAt(date = new Date()): this {
    this.completedAt = date;
    return this;
  }

  withWebhook(url: string): this {
    this.webhookUrl = url;
    return this;
  }

  build(): Order {
    return Order.reconstruct({
      id: this.orderId,
      userId: this.userId,
      status: this.status,
      items: this.items.map(i => ({
        id: ulid(),
        productId: i.productId,
        quantity: i.quantity,
        price: Money.of(i.unitPrice, 'JPY'),
      })),
      createdAt: this.createdAt,
      completedAt: this.completedAt,
      webhookUrl: this.webhookUrl,
    });
  }

  // よく使うプリセット
  static draft(): OrderBuilder {
    return new OrderBuilder().withStatus('draft');
  }

  static submitted(): OrderBuilder {
    return new OrderBuilder().withStatus('pending_payment');
  }

  static completed(): OrderBuilder {
    return new OrderBuilder()
      .withStatus('completed')
      .withCompletedAt(new Date());
  }

  static cancelled(): OrderBuilder {
    return new OrderBuilder().withStatus('cancelled');
  }
}

// ショートカット関数
export const buildOrder = (overrides?: Partial<{
  id: string;
  userId: string;
  status: OrderStatus;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
}>) => {
  let builder = new OrderBuilder();
  if (overrides?.id) builder = builder.withId(overrides.id);
  if (overrides?.userId) builder = builder.withUser(overrides.userId);
  if (overrides?.status) builder = builder.withStatus(overrides.status);
  if (overrides?.items) builder = builder.withItems(overrides.items);
  return builder.build();
};
```

```typescript
// src/__tests__/builders/userBuilder.ts — faker.js統合ビルダー

export class UserBuilder {
  private id = ulid();
  private email = faker.internet.email();
  private name = faker.person.fullName();
  private role: UserRole = 'customer';
  private createdAt = faker.date.past();
  private isVerified = true;
  private stripeCustomerId: string | undefined = undefined;

  withId(id: string): this { this.id = id; return this; }
  withEmail(email: string): this { this.email = email; return this; }
  withName(name: string): this { this.name = name; return this; }
  withRole(role: UserRole): this { this.role = role; return this; }
  asAdmin(): this { return this.withRole('admin'); }
  asPremium(): this { return this.withRole('premium_customer'); }
  unverified(): this { this.isVerified = false; return this; }
  withStripe(customerId = `cus_${faker.string.alphanumeric(14)}`): this {
    this.stripeCustomerId = customerId;
    return this;
  }

  build(): User {
    return User.reconstruct({
      id: this.id,
      email: this.email,
      name: this.name,
      role: this.role,
      createdAt: this.createdAt,
      isVerified: this.isVerified,
      stripeCustomerId: this.stripeCustomerId,
    });
  }

  // DBに保存する場合（リポジトリと組み合わせ）
  async persist(repo: IUserRepository): Promise<User> {
    const user = this.build();
    await repo.save(user);
    return user;
  }
}

export const buildUser = (overrides?: Partial<Parameters<UserBuilder['build']>[0]>) => {
  let builder = new UserBuilder();
  // overridesを適用...
  return builder.build();
};
```

```typescript
// テストでの使用例（宣言的で読みやすい）

describe('OrderService', () => {
  let orderRepo: InMemoryOrderRepository;
  let userRepo: InMemoryUserRepository;
  let service: OrderService;

  beforeEach(() => {
    orderRepo = new InMemoryOrderRepository();
    userRepo = new InMemoryUserRepository();
    service = new OrderService(orderRepo, userRepo);
  });

  // ❌ ビルダーなし（何をテストしているか分かりにくい）
  it('注文をキャンセルできる', async () => {
    const user = User.reconstruct({
      id: 'user-123',
      email: 'test@example.com',
      name: 'テスト太郎',
      role: 'customer',
      createdAt: new Date('2024-01-01'),
      isVerified: true,
      stripeCustomerId: undefined,
    });
    await userRepo.save(user);

    const order = Order.reconstruct({
      id: 'order-456',
      userId: 'user-123',
      status: 'draft',
      items: [{ id: 'item-1', productId: 'prod-1', quantity: 2, price: Money.of(1000, 'JPY') }],
      createdAt: new Date(),
      completedAt: undefined,
      webhookUrl: undefined,
    });
    await orderRepo.save(order);

    await service.cancel('order-456', 'user-123');
    const updated = await orderRepo.findById('order-456');
    expect(updated?.status).toBe('cancelled');
  });

  // ✅ ビルダーあり（テストの意図が明確）
  it('注文をキャンセルできる', async () => {
    const user = await new UserBuilder().withId('user-1').persist(userRepo);
    const order = await OrderBuilder.draft().withUser(user.id).withId('order-1').persistTo(orderRepo);

    await service.cancel(order.id, user.id);

    const updated = await orderRepo.findById(order.id);
    expect(updated?.status).toBe('cancelled');
  });

  it('完了済み注文はキャンセルできない', async () => {
    const user = await new UserBuilder().persist(userRepo);
    const order = await OrderBuilder.completed().withUser(user.id).persistTo(orderRepo);

    await expect(service.cancel(order.id, user.id)).rejects.toThrow(DomainError);
  });

  it('プレミアムユーザーは大量注文ができる', async () => {
    const user = await new UserBuilder().asPremium().persist(userRepo);
    const order = new OrderBuilder()
      .withUser(user.id)
      .withItems(Array.from({ length: 50 }, (_, i) => ({
        productId: `prod-${i}`,
        quantity: 2,
        unitPrice: 1000,
      })))
      .build();

    await expect(service.place(order)).resolves.not.toThrow();
  });
});
```

---

## まとめ

Claude Codeでテストデータビルダーを設計する：

1. **CLAUDE.md** にBuilderはデフォルト値内蔵・メソッドチェーンで部分上書き・`src/__tests__/builders/`に集約・プリセット（`OrderBuilder.completed()`）でよく使うパターンを名前付き提供を明記
2. **デフォルト値内蔵** でテストのセットアップを最小化——`new OrderBuilder().build()`で動くオブジェクトが生成される。テストで重要な部分（status='completed'）だけを明示的に設定できる
3. **`OrderBuilder.completed()`プリセット** でテストの意図を名前で表現——`buildOrder({ status: 'completed', completedAt: new Date() })`より`OrderBuilder.completed()`の方が「完了済み注文でのテスト」という意図が明確
4. **faker.js統合** でランダムだが有効なデータを自動生成——`email = faker.internet.email()`でテストが特定のメアドに依存しなくなる。テストが独立し並列実行してもデータが衝突しない

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
