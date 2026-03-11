---
title: "Claude CodeでTypeScript Branded Typesを設計する：名義型・型安全ID・単位ミス防止"
emoji: "🏷️"
type: "tech"
topics: ["claudecode", "typescript", "nodejs", "security"]
published: true
published_at: "2026-03-14 16:00"
---

## はじめに

`userId`に`productId`を渡してしまうバグ——TypeScriptは構造的型付けなので`string`は`string`として通ってしまう。Branded Types（名義型）で型レベルの混同を防ぐ設計をClaude Codeに生成させる。

---

## CLAUDE.mdにBranded Types設計ルールを書く

```markdown
## Branded Types設計ルール

### ID型定義
- 全エンティティのIDはBranded Typeを使用
- UserId, ProductId, OrderId等を明確に区別
- DB値（Prisma）からのキャスト: as UserId（境界で1回のみ）

### 単位付き数値
- 金額: Money（centesimal整数）、Currency付き
- 距離, 時間: 単位をブランドに含める
- number同士の混在による計算ミスを型で防止

### バリデーション統合
- 外部入力（API）はZodでparse → Branded Typeにキャスト
- DB値は信頼された値としてasキャスト（Prisma型から）
```

---

## Branded Types実装の生成

```
TypeScript Branded Types設計を生成してください。

要件：
- エンティティIDの型安全化
- 金額・単位の型安全計算
- Zodバリデーション統合
- Prisma型との連携

生成ファイル: src/types/branded/
```

---

## 生成されるBranded Types実装

```typescript
// src/types/branded/core.ts — 基底型

// Branded Type の実装（Phantom Typeパターン）
type Branded<T, Brand extends string> = T & { readonly __brand: Brand };

// 型安全なキャスト関数
function brand<T, Brand extends string>(value: T): Branded<T, Brand> {
  return value as Branded<T, Brand>;
}

// =========================
// エンティティID型
// =========================
export type UserId    = Branded<string, 'UserId'>;
export type ProductId = Branded<string, 'ProductId'>;
export type OrderId   = Branded<string, 'OrderId'>;
export type TenantId  = Branded<string, 'TenantId'>;
export type SessionId = Branded<string, 'SessionId'>;

// ID生成ユーティリティ
export const UserId    = { create: (): UserId    => brand(crypto.randomUUID()) };
export const ProductId = { create: (): ProductId => brand(crypto.randomUUID()) };
export const OrderId   = { create: (): OrderId   => brand(crypto.randomUUID()) };

// =========================
// 金額型（通貨別）
// =========================
type CurrencyCode = 'JPY' | 'USD' | 'EUR';
export type Money<C extends CurrencyCode = CurrencyCode> = Branded<number, `Money_${C}`>;
export type JPY = Money<'JPY'>;
export type USD = Money<'USD'>;

export const Money = {
  jpy: (amount: number): JPY => {
    if (!Number.isInteger(amount)) throw new Error('JPY must be integer');
    if (amount < 0) throw new Error('Money cannot be negative');
    return brand(amount);
  },
  usd: (cents: number): USD => {
    if (!Number.isInteger(cents)) throw new Error('USD cents must be integer');
    return brand(cents);
  },
  add: <C extends CurrencyCode>(a: Money<C>, b: Money<C>): Money<C> =>
    brand(a + b),
  multiply: <C extends CurrencyCode>(amount: Money<C>, factor: number): Money<C> =>
    brand(Math.round(amount * factor)),
};

// =========================
// 検証済みメールアドレス型
// =========================
export type ValidEmail = Branded<string, 'ValidEmail'>;

export const ValidEmail = {
  parse: (raw: string): ValidEmail => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(raw)) throw new Error(`Invalid email: ${raw}`);
    return brand(raw.toLowerCase().trim());
  },
};

// =========================
// Hashed値（平文を誤って使わない）
// =========================
export type HashedPassword = Branded<string, 'HashedPassword'>;
export type HashedToken    = Branded<string, 'HashedToken'>;

export const HashedPassword = {
  hash: async (plain: string): Promise<HashedPassword> => {
    const hashed = await bcrypt.hash(plain, 12);
    return brand(hashed);
  },
  verify: async (plain: string, hashed: HashedPassword): Promise<boolean> =>
    bcrypt.compare(plain, hashed),
};
```

```typescript
// src/types/branded/zod.ts — Zod統合

import { z } from 'zod';
import { UserId, ProductId, OrderId, ValidEmail } from './core';

// Zodスキーマ → Branded Type
export const UserIdSchema    = z.string().uuid().transform(id => id as UserId);
export const ProductIdSchema = z.string().uuid().transform(id => id as ProductId);
export const OrderIdSchema   = z.string().uuid().transform(id => id as OrderId);
export const EmailSchema     = z.string().email().transform(e => ValidEmail.parse(e));

// API入力バリデーションスキーマ
export const CreateOrderSchema = z.object({
  userId: UserIdSchema,
  items: z.array(z.object({
    productId: ProductIdSchema,
    quantity: z.number().int().min(1).max(100),
  })).min(1),
});

export type CreateOrderInput = z.infer<typeof CreateOrderSchema>;
```

```typescript
// src/types/branded/prisma.ts — PrismaのDB値からBranded Typeへ変換

import { User, Product, Order } from '@prisma/client';
import { UserId, ProductId, OrderId, ValidEmail, HashedPassword } from './core';

// PrismaモデルをBranded Type付きドメインオブジェクトに変換
export type DomainUser = Omit<User, 'id' | 'email' | 'passwordHash'> & {
  id: UserId;
  email: ValidEmail;
  passwordHash: HashedPassword;
};

export function toDomainUser(prismaUser: User): DomainUser {
  return {
    ...prismaUser,
    id: prismaUser.id as UserId,
    email: prismaUser.email as ValidEmail,   // DBの値は信頼済み
    passwordHash: prismaUser.passwordHash as HashedPassword,
  };
}

// 使用例: 型安全なサービス層
export class OrderService {
  async createOrder(input: CreateOrderInput): Promise<{ orderId: OrderId }> {
    // input.userId はUserId型、input.items[].productId はProductId型
    // 型が違えばコンパイルエラー！

    // これはコンパイルエラー（ProductId を UserId として使えない）:
    // const wrong: UserId = input.items[0].productId; // ❌ エラー

    const order = await prisma.order.create({
      data: {
        userId: input.userId,  // string に変換して保存
        status: 'PENDING',
        items: { create: input.items.map(item => ({
          productId: item.productId,
          quantity: item.quantity,
        }))},
      },
    });

    return { orderId: order.id as OrderId };
  }
}
```

```typescript
// src/types/branded/index.ts — 実際の使用例で型安全の威力を示す

// ❌ Branded Typeなし（型安全でない）
function sendEmailBad(userId: string, recipientId: string): void {
  // userId と recipientId を取り違えても TypeScript は気づかない
}

// ✅ Branded Typeあり（型安全）
function sendEmailSafe(userId: UserId, recipientEmail: ValidEmail): void {
  // 間違ったIDや未検証メールアドレスを渡すとコンパイルエラー
}

// 金額計算の型安全例
const price: JPY = Money.jpy(1000);
const tax:   JPY = Money.jpy(100);
const total: JPY = Money.add(price, tax); // ✅ OK

// const wrong: JPY = Money.add(price, usdAmount); // ❌ コンパイルエラー
```

---

## まとめ

Claude CodeでTypeScript Branded Typesを設計する：

1. **CLAUDE.md** に全エンティティIDをBranded Type・金額はMoney<Currency>・外部入力はZodでparse後キャストを明記
2. **Branded型の構造** `T & { readonly __brand: Brand }` でランタイムオーバーヘッドゼロ、型情報のみ追加
3. **Zodトランスフォーム** でAPIバリデーションと同時にBranded Typeへ変換——境界での一度だけのキャスト
4. **Money.add()の型パラメータ** で通貨の異なる金額を加算しようとするとコンパイルエラー

---

*型設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
