---
title: "Claude Codeでスペシフィケーションパターンを設計する：ビジネスルールの型安全な表現・AND/OR合成・クエリ変換"
emoji: "📐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 19:00"
---

## はじめに

「ビジネスルールがサービス層に散らばっていて、同じ条件を複数の場所で書いている」——スペシフィケーションパターンでビジネスルールをオブジェクトとして表現し、AND/OR合成・テスト容易性・クエリ変換を実現する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにスペシフィケーション設計ルールを書く

```markdown
## スペシフィケーションパターン設計ルール

### スペック設計
- 各スペックは単一のビジネスルールを表現
- `isSatisfiedBy(entity): boolean` で検証
- `toQuery(): WhereClause` でDBクエリに変換（in-memory判定とDB判定を統一）

### 合成
- `spec1.and(spec2)`: 両方を満たす
- `spec1.or(spec2)`: どちらかを満たす
- `spec.not()`: 条件を反転

### 名前付きスペック
- ビジネス用語に対応した名前（例: `EligibleForDiscount`, `PremiumUser`）
- ドメイン語彙でドキュメント化される
```

---

## スペシフィケーション実装の生成

```
スペシフィケーションパターンを設計してください。

要件：
- 基底Specificationクラス
- AND/OR/NOT合成
- Prismaクエリへの変換
- 複数スペックの組み合わせ

生成ファイル: src/domain/specifications/
```

---

## 生成されるスペシフィケーション実装

```typescript
// src/domain/specifications/specification.ts — 基底Specification

export type WhereClause = Record<string, unknown>;

export abstract class Specification<T> {
  abstract isSatisfiedBy(entity: T): boolean;
  abstract toQuery(): WhereClause;  // Prismaのwhere句に変換

  and(other: Specification<T>): Specification<T> {
    return new AndSpecification(this, other);
  }

  or(other: Specification<T>): Specification<T> {
    return new OrSpecification(this, other);
  }

  not(): Specification<T> {
    return new NotSpecification(this);
  }
}

class AndSpecification<T> extends Specification<T> {
  constructor(private readonly left: Specification<T>, private readonly right: Specification<T>) {
    super();
  }

  isSatisfiedBy(entity: T): boolean {
    return this.left.isSatisfiedBy(entity) && this.right.isSatisfiedBy(entity);
  }

  toQuery(): WhereClause {
    return { AND: [this.left.toQuery(), this.right.toQuery()] };
  }
}

class OrSpecification<T> extends Specification<T> {
  constructor(private readonly left: Specification<T>, private readonly right: Specification<T>) {
    super();
  }

  isSatisfiedBy(entity: T): boolean {
    return this.left.isSatisfiedBy(entity) || this.right.isSatisfiedBy(entity);
  }

  toQuery(): WhereClause {
    return { OR: [this.left.toQuery(), this.right.toQuery()] };
  }
}

class NotSpecification<T> extends Specification<T> {
  constructor(private readonly spec: Specification<T>) {
    super();
  }

  isSatisfiedBy(entity: T): boolean {
    return !this.spec.isSatisfiedBy(entity);
  }

  toQuery(): WhereClause {
    return { NOT: this.spec.toQuery() };
  }
}
```

```typescript
// src/domain/specifications/orderSpecs.ts — 注文のスペック定義

interface Order {
  id: string;
  status: string;
  totalAmount: number;
  createdAt: Date;
  userId: string;
  isPaid: boolean;
  hasInternationalShipping: boolean;
}

// スペック: 確定済み注文
export class ConfirmedOrderSpec extends Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return order.status === 'confirmed';
  }
  toQuery(): WhereClause {
    return { status: 'confirmed' };
  }
}

// スペック: 高額注文（5万円以上）
export class HighValueOrderSpec extends Specification<Order> {
  constructor(private readonly threshold: number = 50_000) {
    super();
  }

  isSatisfiedBy(order: Order): boolean {
    return order.totalAmount >= this.threshold;
  }

  toQuery(): WhereClause {
    return { totalAmount: { gte: this.threshold } };
  }
}

// スペック: 最近の注文（30日以内）
export class RecentOrderSpec extends Specification<Order> {
  constructor(private readonly daysAgo: number = 30) {
    super();
  }

  isSatisfiedBy(order: Order): boolean {
    const cutoff = new Date(Date.now() - this.daysAgo * 86400_000);
    return order.createdAt >= cutoff;
  }

  toQuery(): WhereClause {
    return { createdAt: { gte: new Date(Date.now() - this.daysAgo * 86400_000) } };
  }
}

// スペック: 海外配送あり
export class InternationalOrderSpec extends Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return order.hasInternationalShipping;
  }
  toQuery(): WhereClause {
    return { hasInternationalShipping: true };
  }
}

// スペック: 支払済み
export class PaidOrderSpec extends Specification<Order> {
  isSatisfiedBy(order: Order): boolean {
    return order.isPaid;
  }
  toQuery(): WhereClause {
    return { isPaid: true };
  }
}

// 合成スペック: 優先処理が必要な注文（高額 かつ 確定済み かつ 未払い）
export const PriorityOrderSpec = new HighValueOrderSpec(50_000)
  .and(new ConfirmedOrderSpec())
  .and(new PaidOrderSpec().not());

// 合成スペック: レポート対象（直近30日の確定済み注文 または 5万円以上の全確定済み注文）
export const ReportableOrderSpec = new RecentOrderSpec(30)
  .and(new ConfirmedOrderSpec())
  .or(new HighValueOrderSpec(50_000).and(new ConfirmedOrderSpec()));
```

```typescript
// src/domain/specifications/orderRepository.ts — スペックを使ったリポジトリ

export class OrderRepository {
  // スペックでフィルタ（DB側のクエリに変換）
  async findBySpec(spec: Specification<Order>): Promise<Order[]> {
    return prisma.order.findMany({
      where: spec.toQuery(), // Prismaのwhere句として使用
    });
  }

  // スペックで検証（in-memory）
  filterInMemory(orders: Order[], spec: Specification<Order>): Order[] {
    return orders.filter(o => spec.isSatisfiedBy(o));
  }
}

// 使用例
const repo = new OrderRepository();

// DB側でフィルタ（効率的）
const priorityOrders = await repo.findBySpec(PriorityOrderSpec);

// 海外配送付きの高額確定済み注文
const internationalHighValue = await repo.findBySpec(
  new InternationalOrderSpec()
    .and(new HighValueOrderSpec(100_000))
    .and(new ConfirmedOrderSpec())
);

// クーポン対象のチェック（in-memory validation）
function isEligibleForCoupon(order: Order): boolean {
  const eligibilitySpec = new RecentOrderSpec(90)
    .and(new PaidOrderSpec())
    .and(new HighValueOrderSpec(10_000));

  return eligibilitySpec.isSatisfiedBy(order);
}

// ビジネスルールをドキュメント化（スペック名がそのままルール説明）
const DISCOUNT_ELIGIBILITY_SPEC = new PaidOrderSpec()
  .and(new HighValueOrderSpec(30_000))
  .and(new RecentOrderSpec(60));

// テスト（スペック単独でテスト可能）
describe('HighValueOrderSpec', () => {
  it('should satisfy orders above threshold', () => {
    const spec = new HighValueOrderSpec(50_000);
    const order = { totalAmount: 60_000 } as Order;
    expect(spec.isSatisfiedBy(order)).toBe(true);
  });

  it('should generate correct Prisma query', () => {
    const spec = new HighValueOrderSpec(50_000);
    expect(spec.toQuery()).toEqual({ totalAmount: { gte: 50_000 } });
  });
});
```

---

## まとめ

Claude Codeでスペシフィケーションパターンを設計する：

1. **CLAUDE.md** に各スペックは単一ビジネスルール・isSatisfiedBy（in-memory）とtoQuery（Prisma変換）を両方実装・AND/OR/NOT合成でルールを組み合わせを明記
2. **`toQuery()`でPrismaのwhere句に変換** することでin-memoryとDBの判定ロジックを統一——同じスペックが「ドメインオブジェクトの検証」と「DBクエリ生成」の両方に使える
3. **スペックの合成** で「高額 AND 確定済み AND 未払い」という複合ルールを読みやすく表現——if文のネストではなくスペックオブジェクトの組み合わせでビジネスルールをドキュメント化
4. **スペック単体テスト** で各ビジネスルールを独立して検証——`HighValueOrderSpec.isSatisfiedBy({ totalAmount: 60000 })`のような純粋関数のテストが書ける

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
