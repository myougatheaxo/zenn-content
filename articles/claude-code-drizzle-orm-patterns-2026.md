---
title: "Drizzle ORM実践パターン：型安全なDBアクセスの設計"
emoji: "🌧️"
type: "tech"
topics:
  - claudecode
  - typescript
  - drizzle
  - database
published: true
---

## Drizzle ORMとは何か

Drizzle ORMは、TypeScriptファーストの軽量ORMだ。Prismaと比較して「よりSQLに近い」という特徴があり、型推論が強力でバンドルサイズが小さい。Edge Runtime（Cloudflare Workers等）でも動作する。

Prismaとの主な違い:

| 比較点 | Prisma | Drizzle |
|--------|--------|---------|
| スキーマ | `.prisma`ファイル | TypeScript |
| 型推論 | 生成コード依存 | 推論自動 |
| Edge対応 | 制限あり | フル対応 |
| バンドルサイズ | 大きい | 小さい |
| SQL近接性 | 高抽象化 | SQLに近い |

「Drizzleのクエリを書くことはSQLを書くことだ」という思想が根底にある。

## スキーマ定義：TypeScriptでテーブルを定義する

```typescript
// src/db/schema.ts
import {
  pgTable, serial, text, varchar, integer,
  timestamp, boolean, decimal, jsonb, index,
  uniqueIndex, pgEnum,
} from "drizzle-orm/pg-core";
import { relations } from "drizzle-orm";

export const roleEnum = pgEnum("role", ["admin", "user", "viewer"]);

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  name: varchar("name", { length: 255 }).notNull(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  role: roleEnum("role").default("user").notNull(),
  metadata: jsonb("metadata").$type<Record<string, unknown>>().default({}),
  isActive: boolean("is_active").default(true).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
}, (table) => ({
  emailIdx: uniqueIndex("email_idx").on(table.email),
  roleIdx: index("role_idx").on(table.role),
}));

export const posts = pgTable("posts", {
  id: serial("id").primaryKey(),
  title: text("title").notNull(),
  content: text("content").notNull(),
  authorId: integer("author_id").notNull().references(() => users.id, {
    onDelete: "cascade",
  }),
  publishedAt: timestamp("published_at", { withTimezone: true }),
  viewCount: integer("view_count").default(0).notNull(),
  price: decimal("price", { precision: 10, scale: 2 }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// リレーション定義
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
}));

// 型のエクスポート
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
export type Post = typeof posts.$inferSelect;
export type NewPost = typeof posts.$inferInsert;
```

## データベース接続とクエリの基本

```typescript
// src/db/index.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30_000,
});

export const db = drizzle(pool, { schema });

// 基本的なCRUD操作
import { eq, and, or, desc, gte, like, sql } from "drizzle-orm";

// SELECT
async function findUser(id: number): Promise<User | undefined> {
  const result = await db.select().from(users).where(eq(users.id, id)).limit(1);
  return result[0];
}

// SELECT with JOIN（リレーションを使った方法）
async function findPostsWithAuthor(limit = 10) {
  return await db.query.posts.findMany({
    with: { author: true },
    orderBy: desc(posts.createdAt),
    limit,
  });
}

// INSERT
async function createUser(data: NewUser): Promise<User> {
  const [user] = await db.insert(users).values(data).returning();
  return user;
}

// UPDATE
async function updateUser(id: number, data: Partial<NewUser>): Promise<User | undefined> {
  const [updated] = await db
    .update(users)
    .set({ ...data, updatedAt: new Date() })
    .where(eq(users.id, id))
    .returning();
  return updated;
}

// DELETE
async function deleteUser(id: number): Promise<boolean> {
  const result = await db.delete(users).where(eq(users.id, id));
  return result.rowCount > 0;
}
```

## 複雑なクエリパターン

```typescript
// フィルタリングの動的構築
interface UserFilter {
  role?: "admin" | "user" | "viewer";
  isActive?: boolean;
  search?: string;
  createdAfter?: Date;
}

async function findUsers(filter: UserFilter, page = 1, pageSize = 20) {
  const conditions = [];

  if (filter.role) {
    conditions.push(eq(users.role, filter.role));
  }
  if (filter.isActive !== undefined) {
    conditions.push(eq(users.isActive, filter.isActive));
  }
  if (filter.search) {
    conditions.push(
      or(
        like(users.name, `%${filter.search}%`),
        like(users.email, `%${filter.search}%`),
      )
    );
  }
  if (filter.createdAfter) {
    conditions.push(gte(users.createdAt, filter.createdAfter));
  }

  const whereClause = conditions.length > 0 ? and(...conditions) : undefined;

  // ページネーション付きクエリ
  const [data, countResult] = await Promise.all([
    db
      .select()
      .from(users)
      .where(whereClause)
      .orderBy(desc(users.createdAt))
      .limit(pageSize)
      .offset((page - 1) * pageSize),
    db
      .select({ count: sql<number>`count(*)::int` })
      .from(users)
      .where(whereClause),
  ]);

  return {
    data,
    total: countResult[0].count,
    page,
    pageSize,
    totalPages: Math.ceil(countResult[0].count / pageSize),
  };
}

// サブクエリ
async function findUsersWithPostCount() {
  return await db
    .select({
      id: users.id,
      name: users.name,
      postCount: sql<number>`count(${posts.id})::int`,
    })
    .from(users)
    .leftJoin(posts, eq(users.id, posts.authorId))
    .groupBy(users.id, users.name)
    .orderBy(desc(sql`count(${posts.id})`));
}
```

## トランザクション

```typescript
// トランザクション内での複数操作
async function transferCredits(
  fromUserId: number,
  toUserId: number,
  amount: number,
): Promise<void> {
  await db.transaction(async (tx) => {
    // 送信元の残高確認
    const [sender] = await tx
      .select({ credits: users.metadata })
      .from(users)
      .where(eq(users.id, fromUserId))
      .for("update");  // SELECT FOR UPDATE

    const currentCredits = (sender.credits as any)?.credits ?? 0;
    if (currentCredits < amount) {
      throw new Error("Insufficient credits");
    }

    // 残高更新（両方）
    await tx
      .update(users)
      .set({
        metadata: sql`jsonb_set(metadata, '{credits}', to_jsonb(${currentCredits - amount}))`,
      })
      .where(eq(users.id, fromUserId));

    await tx
      .update(users)
      .set({
        metadata: sql`jsonb_set(
          COALESCE(metadata, '{}'),
          '{credits}',
          to_jsonb(COALESCE((metadata->>'credits')::int, 0) + ${amount})
        )`,
      })
      .where(eq(users.id, toUserId));
  });
}
```

## マイグレーション管理

```typescript
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle/migrations",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

```bash
# マイグレーション生成
npx drizzle-kit generate

# マイグレーション適用
npx drizzle-kit migrate

# 現在のスキーマとDBの差分確認
npx drizzle-kit check

# Drizzle Studio（GUIブラウザ）
npx drizzle-kit studio
```

## Edge環境（Cloudflare Workers）での使用

```typescript
// Cloudflare Workers + D1 + Drizzle
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

export default {
  async fetch(req: Request, env: { DB: D1Database }): Promise<Response> {
    const db = drizzle(env.DB, { schema });

    const allUsers = await db.select().from(schema.users).limit(10);
    return Response.json(allUsers);
  },
};
```

DrizzleはSQLの知識をそのまま活かしつつ、TypeScriptの型安全性を得られる。「ORMのブラックボックスに隠れたSQLが怖い」という経験があるなら、Drizzleは非常に相性が良い選択肢だ。

---

この記事の内容は、Claude Code完全攻略ガイド（全7章）の一部。CLAUDE.md設計、Hooks実践、MCPセットアップ、マルチエージェント構成まで全7章にまとめた完全版はnoteで公開している。

https://note.com/myougatheaxo/n/ncf4f814d6b68

*みょうが (@myougaTheAxo) ― ウーパールーパーのVTuber。AIツールの実践的な使い方を発信中。*
