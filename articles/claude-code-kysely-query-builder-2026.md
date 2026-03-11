---
title: "Claude CodeでKyselyタイプセーフクエリビルダーを設計する：複雑なJOIN・動的フィルタ・Prisma移行"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "prisma"]
published: true
published_at: "2026-03-15 16:00"
---

## はじめに

生SQLは型安全でなく、ORMは複雑なJOINが苦手——KyselyはTypeScriptの型システムを活用した型安全クエリビルダーで、複雑なSQLも型チェックされる。Claude Codeに設計させる。

---

## CLAUDE.mdにKysely設計ルールを書く

```markdown
## Kyselyクエリビルダー設計ルール

### 採用基準
- 複雑なJOIN・サブクエリ・ウィンドウ関数: Kysely
- シンプルなCRUD: Prisma（DX優先）
- 生SQL: 禁止（型安全でないため）

### スキーマ型定義
- Database型インターフェースで全テーブルを定義
- Insertable<T>, Selectable<T>, Updateable<T>を使い分ける
- Generated<T>でauto-incrementカラムを区別

### 動的クエリ
- 条件付きフィルタ: .where()を動的に追加
- 型安全な動的ソート: OrderByExpression型を使用
- ページネーション: .offset().limit()で統一
```

---

## Kysely実装の生成

```
Kyselyを使った型安全クエリビルダーシステムを設計してください。

要件：
- DB型定義
- 複雑なJOINクエリ
- 動的フィルタ
- ウィンドウ関数
- Prismaとの共存

生成ファイル: src/db/kysely/
```

---

## 生成されるKysely実装

```typescript
// src/db/kysely/types.ts — DB型定義

import { Generated, Insertable, Selectable, Updateable } from 'kysely';

// テーブルごとの型定義
export interface UserTable {
  id: Generated<string>;  // DB自動生成
  email: string;
  name: string;
  role: 'user' | 'admin';
  tenant_id: string;
  created_at: Generated<Date>;
  updated_at: Generated<Date>;
  deleted_at: Date | null;
}

export interface OrderTable {
  id: Generated<string>;
  user_id: string;
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  total_amount: number;
  created_at: Generated<Date>;
}

export interface OrderItemTable {
  id: Generated<string>;
  order_id: string;
  product_id: string;
  quantity: number;
  unit_price: number;
}

export interface ProductTable {
  id: Generated<string>;
  name: string;
  price: number;
  category: string;
  inventory: number;
  created_at: Generated<Date>;
}

// Database型（Kyselyに渡す）
export interface Database {
  users: UserTable;
  orders: OrderTable;
  order_items: OrderItemTable;
  products: ProductTable;
}

// 操作別の型エイリアス
export type User = Selectable<UserTable>;
export type NewUser = Insertable<UserTable>;
export type UserUpdate = Updateable<UserTable>;
```

```typescript
// src/db/kysely/client.ts — Kyselyインスタンス

import { Kysely, PostgresDialect } from 'kysely';
import { Pool } from 'pg';
import { Database } from './types';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
});

export const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
  log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
});
```

```typescript
// src/db/kysely/queries/orderAnalytics.ts — 複雑なJOINとウィンドウ関数

export async function getOrderAnalytics(tenantId: string, month: string): Promise<OrderAnalytics[]> {
  return db
    .selectFrom('orders as o')
    .innerJoin('users as u', 'u.id', 'o.user_id')
    .innerJoin('order_items as oi', 'oi.order_id', 'o.id')
    .innerJoin('products as p', 'p.id', 'oi.product_id')
    .select([
      'u.id as userId',
      'u.name as userName',
      'p.category',
      db.fn.sum('oi.unit_price').multiply(db.ref('oi.quantity')).as('categoryRevenue'),
      db.fn.count('o.id').as('orderCount'),
      // ウィンドウ関数: ユーザーごとのカテゴリ別売上ランキング
      db.fn
        .agg<number>('rank')
        .over(ob => ob.partitionBy('u.id').orderBy(
          db.fn.sum('oi.unit_price').multiply(db.ref('oi.quantity')),
          'desc'
        ))
        .as('categoryRank'),
    ])
    .where('u.tenant_id', '=', tenantId)
    .where(db.fn('date_trunc', ['month', db.ref('o.created_at')]), '=', new Date(month))
    .where('o.status', '!=', 'cancelled')
    .groupBy(['u.id', 'u.name', 'p.category'])
    .orderBy('userId')
    .orderBy('categoryRevenue', 'desc')
    .execute();
}

// 動的フィルタクエリ（型安全）
interface OrderFilterParams {
  userId?: string;
  status?: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  minAmount?: number;
  maxAmount?: number;
  fromDate?: Date;
  toDate?: Date;
  sortBy?: 'created_at' | 'total_amount';
  sortOrder?: 'asc' | 'desc';
  page?: number;
  limit?: number;
}

export async function findOrders(params: OrderFilterParams): Promise<{
  items: Order[];
  total: number;
}> {
  const limit = params.limit ?? 20;
  const offset = ((params.page ?? 1) - 1) * limit;

  // ベースクエリ
  let query = db
    .selectFrom('orders')
    .selectAll()
    .where('deleted_at', 'is', null);

  // 動的条件追加（型安全）
  if (params.userId) query = query.where('user_id', '=', params.userId);
  if (params.status) query = query.where('status', '=', params.status);
  if (params.minAmount) query = query.where('total_amount', '>=', params.minAmount);
  if (params.maxAmount) query = query.where('total_amount', '<=', params.maxAmount);
  if (params.fromDate) query = query.where('created_at', '>=', params.fromDate);
  if (params.toDate) query = query.where('created_at', '<=', params.toDate);

  // ソート
  const sortBy = params.sortBy ?? 'created_at';
  const sortOrder = params.sortOrder ?? 'desc';
  query = query.orderBy(sortBy, sortOrder);

  // 件数取得とデータ取得を並列実行
  const [items, countResult] = await Promise.all([
    query.limit(limit).offset(offset).execute(),
    query.select(db.fn.countAll<number>().as('count')).executeTakeFirstOrThrow(),
  ]);

  return { items, total: Number(countResult.count) };
}
```

```typescript
// src/db/kysely/queries/userReport.ts — CTEを使った複雑なレポートクエリ

export async function getUserActivityReport(userId: string): Promise<UserActivityReport> {
  return db
    .with('order_stats', (qb) =>
      qb
        .selectFrom('orders')
        .select([
          'user_id',
          db.fn.count('id').as('totalOrders'),
          db.fn.sum('total_amount').as('totalRevenue'),
          db.fn.max('created_at').as('lastOrderAt'),
        ])
        .where('user_id', '=', userId)
        .where('status', '!=', 'cancelled')
        .groupBy('user_id')
    )
    .with('category_breakdown', (qb) =>
      qb
        .selectFrom('orders as o')
        .innerJoin('order_items as oi', 'oi.order_id', 'o.id')
        .innerJoin('products as p', 'p.id', 'oi.product_id')
        .select([
          'p.category',
          db.fn.sum(db.ref('oi.unit_price').multiply(db.ref('oi.quantity'))).as('revenue'),
        ])
        .where('o.user_id', '=', userId)
        .groupBy('p.category')
        .orderBy('revenue', 'desc')
        .limit(5)
    )
    .selectFrom('users as u')
    .innerJoin('order_stats as os', 'os.user_id', 'u.id')
    .select([
      'u.id',
      'u.name',
      'u.email',
      'os.totalOrders',
      'os.totalRevenue',
      'os.lastOrderAt',
    ])
    .where('u.id', '=', userId)
    .executeTakeFirstOrThrow();
}
```

---

## まとめ

Claude CodeでKyselyクエリビルダーを設計する：

1. **CLAUDE.md** に複雑JOIN/サブクエリはKysely・シンプルCRUDはPrisma・生SQL禁止という採用基準を明記
2. **Insertable/Selectable/Updateable** で挿入・取得・更新それぞれの型を自動導出——Generated<>フィールドの混同を防止
3. **動的フィルタ** で`if (params.userId) query = query.where(...)` を型安全に積み重ね——文字列結合SQL不要
4. **CTEを使ったレポートクエリ** で複数ステップの集計を可読性高く記述——JOIN地獄を回避

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
