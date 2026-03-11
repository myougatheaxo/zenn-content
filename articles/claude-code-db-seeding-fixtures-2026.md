---
title: "Claude CodeでDBシーディング・テストフィクスチャを設計する：Factory・Faker・Prisma統合"
emoji: "🌱"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "prisma", "testing"]
published: true
published_at: "2026-03-14 15:00"
---

## はじめに

テストのたびに手書きでユーザーを作るのは終わり——FactoryパターンとFaker.jsでリアルなテストデータを生成し、Prismaと統合したシーディングシステムをClaude Codeに設計させる。

---

## CLAUDE.mdにDBシーディング設計ルールを書く

```markdown
## DBシーディング・テストフィクスチャ設計ルール

### Factory設計
- エンティティごとにFactory関数を定義
- デフォルト値はFaker.jsでリアルなデータ
- オーバーライド: create({ name: 'specific name' })で上書き可能
- リレーション: createWithRelations()で関連データも生成

### 環境別シーディング
- development: 開発用デモデータ（管理者・一般ユーザー各5名）
- test: テスト前にクリア→最小限のフィクスチャ
- production: マスターデータのみ（カテゴリ・設定等）

### テスト統合
- 各テストは独立したデータを持つ（beforeEach でFactory使用）
- DBクリーン: deleteMany順序（FK制約の逆順）
- トランザクション内テスト: テスト後にロールバック
```

---

## DBシーディングシステムの生成

```
Prisma + Faker.jsによるDBシーディング・テストフィクスチャを設計してください。

要件：
- エンティティFactory関数
- 環境別シーディング
- テストヘルパー統合
- リレーション対応

生成ファイル: prisma/factories/, prisma/seeds/
```

---

## 生成されるシーディング実装

```typescript
// prisma/factories/index.ts — Factory基底クラス

import { faker } from '@faker-js/faker/locale/ja';
import { PrismaClient } from '@prisma/client';

faker.locale = 'ja'; // 日本語ロケール

export class UserFactory {
  static build(overrides: Partial<UserCreateInput> = {}): UserCreateInput {
    return {
      email: faker.internet.email(),
      name: faker.person.fullName(),
      passwordHash: '$2b$12$hashedpassword', // テスト用固定ハッシュ
      role: 'user',
      avatarUrl: faker.image.avatar(),
      createdAt: faker.date.past({ years: 1 }),
      ...overrides,
    };
  }

  static async create(
    prisma: PrismaClient,
    overrides: Partial<UserCreateInput> = {}
  ) {
    return prisma.user.create({ data: this.build(overrides) });
  }

  static async createMany(
    prisma: PrismaClient,
    count: number,
    overrides: Partial<UserCreateInput> = {}
  ) {
    return Promise.all(
      Array.from({ length: count }, () => this.create(prisma, overrides))
    );
  }

  // 管理者ユーザー専用ファクトリー
  static buildAdmin(overrides: Partial<UserCreateInput> = {}): UserCreateInput {
    return this.build({ role: 'admin', email: `admin-${faker.string.uuid()}@example.com`, ...overrides });
  }
}

export class ProductFactory {
  static build(overrides: Partial<ProductCreateInput> = {}): ProductCreateInput {
    return {
      name: faker.commerce.productName(),
      description: faker.commerce.productDescription(),
      price: parseInt(faker.commerce.price({ min: 100, max: 100000, dec: 0 })),
      category: faker.helpers.arrayElement(['electronics', 'books', 'clothing', 'food']),
      inventory: faker.number.int({ min: 0, max: 500 }),
      imageUrl: faker.image.url(),
      sku: faker.string.alphanumeric(10).toUpperCase(),
      ...overrides,
    };
  }

  static async create(prisma: PrismaClient, overrides: Partial<ProductCreateInput> = {}) {
    return prisma.product.create({ data: this.build(overrides) });
  }
}

export class OrderFactory {
  // 注文はユーザーと商品が必要なので、自動作成オプション付き
  static async create(
    prisma: PrismaClient,
    options: {
      userId?: string;
      productIds?: string[];
      status?: OrderStatus;
    } = {}
  ) {
    // ユーザーが指定されていなければ作成
    const userId = options.userId ?? (await UserFactory.create(prisma)).id;

    // 商品が指定されていなければ作成
    const productIds = options.productIds ?? [
      (await ProductFactory.create(prisma)).id,
    ];

    // 各商品の価格を取得して合計計算
    const products = await prisma.product.findMany({
      where: { id: { in: productIds } },
      select: { id: true, price: true },
    });
    const totalAmount = products.reduce((sum, p) => sum + p.price, 0);

    return prisma.order.create({
      data: {
        userId,
        status: options.status ?? 'PENDING',
        totalAmount,
        items: {
          create: products.map(p => ({
            productId: p.id,
            quantity: faker.number.int({ min: 1, max: 5 }),
            unitPrice: p.price,
          })),
        },
      },
      include: { items: true },
    });
  }
}
```

```typescript
// prisma/seeds/development.ts — 開発用シーディング

import { PrismaClient } from '@prisma/client';
import { UserFactory, ProductFactory, OrderFactory } from '../factories';

const prisma = new PrismaClient();

async function seedDevelopment() {
  console.log('🌱 Seeding development database...');

  // クリーンアップ（FK制約の逆順）
  await prisma.orderItem.deleteMany();
  await prisma.order.deleteMany();
  await prisma.product.deleteMany();
  await prisma.user.deleteMany();

  // 管理者ユーザー
  const admin = await UserFactory.create(prisma, {
    email: 'admin@example.com',
    name: '管理者',
    role: 'admin',
  });
  console.log(`✅ Admin: ${admin.email}`);

  // 一般ユーザー5名
  const users = await UserFactory.createMany(prisma, 5);
  console.log(`✅ Users: ${users.length} created`);

  // 商品50件（カテゴリ別）
  const categories = ['electronics', 'books', 'clothing', 'food'];
  const products = await Promise.all(
    Array.from({ length: 50 }, (_, i) =>
      ProductFactory.create(prisma, {
        category: categories[i % categories.length],
      })
    )
  );
  console.log(`✅ Products: ${products.length} created`);

  // 各ユーザーに注文2-5件
  let orderCount = 0;
  for (const user of users) {
    const count = faker.number.int({ min: 2, max: 5 });
    for (let i = 0; i < count; i++) {
      const selectedProducts = faker.helpers.arrayElements(products, { min: 1, max: 4 });
      await OrderFactory.create(prisma, {
        userId: user.id,
        productIds: selectedProducts.map(p => p.id),
      });
      orderCount++;
    }
  }
  console.log(`✅ Orders: ${orderCount} created`);

  console.log('🎉 Development seed completed');
}

seedDevelopment()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

```typescript
// prisma/seeds/test-helpers.ts — テスト用ヘルパー

import { PrismaClient } from '@prisma/client';
import { UserFactory, ProductFactory, OrderFactory } from '../factories';

const testPrisma = new PrismaClient({
  datasources: { db: { url: process.env.TEST_DATABASE_URL } },
});

// テストスイート共通セットアップ
export async function cleanDatabase(): Promise<void> {
  // FK制約の逆順で削除
  const tables = [
    'orderItem',
    'order',
    'product',
    'refreshToken',
    'session',
    'user',
  ] as const;

  await testPrisma.$transaction(
    tables.map(table => (testPrisma[table] as any).deleteMany())
  );
}

// よく使うフィクスチャをまとめたヘルパー
export async function createTestWorld() {
  const admin = await UserFactory.create(testPrisma, {
    email: 'admin@test.com',
    role: 'admin',
  });
  const user = await UserFactory.create(testPrisma, {
    email: 'user@test.com',
  });
  const product = await ProductFactory.create(testPrisma, {
    price: 1000,
    inventory: 100,
  });
  const order = await OrderFactory.create(testPrisma, {
    userId: user.id,
    productIds: [product.id],
  });

  return { admin, user, product, order, prisma: testPrisma };
}

// Vitest用グローバルセットアップ
export async function setupTestDatabase(): Promise<void> {
  await cleanDatabase();
}

// package.json scripts:
// "test": "vitest run",
// "seed:dev": "ts-node prisma/seeds/development.ts",
// "seed:test": "ts-node prisma/seeds/test.ts",
```

---

## まとめ

Claude CodeでDBシーディング・テストフィクスチャを設計する：

1. **CLAUDE.md** にFactory関数・Faker.jaロケール・FK逆順クリーン・環境別シーディングを明記
2. **build()/create()の分離** でDBを使わない単体テスト（build）とDB統合テスト（create）を使い分け
3. **OrderFactory.create()の自動作成** で関連エンティティが無ければ自動生成——テストコードを簡潔に
4. **createTestWorld()** でよく使うフィクスチャセット（admin/user/product/order）をワンライナーで準備

---

*テスト設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
