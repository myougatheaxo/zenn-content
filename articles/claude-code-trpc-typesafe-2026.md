---
title: "Claude Codeでtアイプセーフ APIを設計する：tRPC・Zod・Prismaエンドツーエンド型安全"
emoji: "🔗"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "trpc", "prisma"]
published: true
published_at: "2026-03-13 11:00"
---

## はじめに

OpenAPI specを書かずにクライアント-サーバー間の型安全を実現する——tRPC + Zod + Prismaでエンドツーエンドの型推論を設計する。Claude Codeに生成させる。

---

## CLAUDE.mdにtRPC設計ルールを書く

```markdown
## tRPC + Zod + Prisma 設計ルール

### tRPCルーター設計
- Zodスキーマで入力バリデーション（実行時+コンパイル時）
- publicProcedure / protectedProcedure の2段階
- protectedProcedureはJWT検証ミドルウェア必須
- エラーはTRPCError（code: 'UNAUTHORIZED', 'NOT_FOUND', 'BAD_REQUEST'）

### Prisma型共有
- PrismaのSelect型をtRPCレスポンスに使用
- パスワードハッシュ等の機密フィールドは明示的に除外
- PaginatedResult<T>ジェネリック型を統一

### クライアント統合
- React Query v5統合（useQuery/useMutation）
- 楽観的更新はonMutate/onError/onSettledパターン
- SSE/WebSocketはtRPCのsubscription
```

---

## tRPC APIの生成

```
tRPC + Zod + Prismaでエンドツーエンド型安全APIを設計してください。

要件：
- ユーザー認証付きCRUD API
- Zodバリデーション
- Prismaとの型統合
- React Query統合
- エラーハンドリング

生成ファイル: src/trpc/
```

---

## 生成されるtRPC実装

```typescript
// src/trpc/context.ts — コンテキスト定義

import { inferAsyncReturnType } from '@trpc/server';
import { prisma } from '../lib/prisma';

export async function createContext({ req }: { req: Request }) {
  // JWTからユーザーを取得（失敗してもnull、認証はprocedureレベルで実施）
  let user: { id: string; role: string } | null = null;

  const token = req.headers.get('authorization')?.replace('Bearer ', '');
  if (token) {
    try {
      const payload = await verifyJWT(token);
      user = { id: payload.sub, role: payload.role };
    } catch {
      // 無効なトークンはnullのまま
    }
  }

  return { prisma, user };
}

export type Context = inferAsyncReturnType<typeof createContext>;
```

```typescript
// src/trpc/init.ts — tRPC初期化

import { initTRPC, TRPCError } from '@trpc/server';
import superjson from 'superjson';
import { Context } from './context';

const t = initTRPC.context<Context>().create({
  transformer: superjson, // Date/Map/Set をJSON変換
});

export const router = t.router;
export const publicProcedure = t.procedure;

// 認証必須プロシージャ
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED', message: 'Authentication required' });
  }
  return next({ ctx: { ...ctx, user: ctx.user } }); // user を non-null に絞り込み
});

// 管理者専用プロシージャ
export const adminProcedure = protectedProcedure.use(({ ctx, next }) => {
  if (ctx.user.role !== 'admin') {
    throw new TRPCError({ code: 'FORBIDDEN', message: 'Admin access required' });
  }
  return next({ ctx });
});
```

```typescript
// src/trpc/routers/posts.ts — 投稿ルーター

import { z } from 'zod';
import { TRPCError } from '@trpc/server';
import { router, publicProcedure, protectedProcedure } from '../init';
import { Prisma } from '@prisma/client';

// 安全な投稿フィールド選択（authorのパスワードを除外）
const postSelect = {
  id: true,
  title: true,
  content: true,
  published: true,
  createdAt: true,
  author: {
    select: { id: true, name: true, email: true }, // passwordHashを除外
  },
} satisfies Prisma.PostSelect;

type PostWithAuthor = Prisma.PostGetPayload<{ select: typeof postSelect }>;

// ページネーション結果型
interface PaginatedResult<T> {
  items: T[];
  nextCursor?: string;
  total: number;
}

export const postsRouter = router({
  // 投稿一覧（カーソルページネーション）
  list: publicProcedure
    .input(z.object({
      cursor: z.string().optional(),
      limit: z.number().min(1).max(100).default(20),
      published: z.boolean().optional(),
    }))
    .query(async ({ ctx, input }): Promise<PaginatedResult<PostWithAuthor>> => {
      const { cursor, limit, published } = input;

      const items = await ctx.prisma.post.findMany({
        where: { published: published ?? true },
        select: postSelect,
        take: limit + 1,
        cursor: cursor ? { id: cursor } : undefined,
        orderBy: { createdAt: 'desc' },
      });

      const hasMore = items.length > limit;
      const nextCursor = hasMore ? items[limit].id : undefined;

      const total = await ctx.prisma.post.count({ where: { published: published ?? true } });

      return { items: items.slice(0, limit), nextCursor, total };
    }),

  // 投稿詳細
  byId: publicProcedure
    .input(z.object({ id: z.string().cuid() }))
    .query(async ({ ctx, input }) => {
      const post = await ctx.prisma.post.findUnique({
        where: { id: input.id },
        select: postSelect,
      });
      if (!post) throw new TRPCError({ code: 'NOT_FOUND', message: 'Post not found' });
      return post;
    }),

  // 投稿作成（認証必須）
  create: protectedProcedure
    .input(z.object({
      title: z.string().min(1).max(200),
      content: z.string().min(1).max(50000),
      published: z.boolean().default(false),
    }))
    .mutation(async ({ ctx, input }) => {
      return ctx.prisma.post.create({
        data: { ...input, authorId: ctx.user.id },
        select: postSelect,
      });
    }),

  // 投稿更新（作成者のみ）
  update: protectedProcedure
    .input(z.object({
      id: z.string().cuid(),
      title: z.string().min(1).max(200).optional(),
      content: z.string().min(1).max(50000).optional(),
      published: z.boolean().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      const { id, ...data } = input;

      // 所有権確認
      const post = await ctx.prisma.post.findUnique({ where: { id } });
      if (!post) throw new TRPCError({ code: 'NOT_FOUND' });
      if (post.authorId !== ctx.user.id && ctx.user.role !== 'admin') {
        throw new TRPCError({ code: 'FORBIDDEN', message: 'Not the post author' });
      }

      return ctx.prisma.post.update({ where: { id }, data, select: postSelect });
    }),

  // 投稿削除
  delete: protectedProcedure
    .input(z.object({ id: z.string().cuid() }))
    .mutation(async ({ ctx, input }) => {
      const post = await ctx.prisma.post.findUnique({ where: { id: input.id } });
      if (!post) throw new TRPCError({ code: 'NOT_FOUND' });
      if (post.authorId !== ctx.user.id && ctx.user.role !== 'admin') {
        throw new TRPCError({ code: 'FORBIDDEN' });
      }
      await ctx.prisma.post.delete({ where: { id: input.id } });
      return { success: true };
    }),
});
```

```typescript
// src/trpc/client.ts — フロントエンドクライアント（React Query統合）

import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '../server/trpc/router';

export const trpc = createTRPCReact<AppRouter>();

// src/App.tsx — プロバイダー設定
export function App() {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: '/api/trpc',
          headers: () => {
            const token = localStorage.getItem('accessToken');
            return token ? { Authorization: `Bearer ${token}` } : {};
          },
        }),
      ],
    })
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        <Router />
      </QueryClientProvider>
    </trpc.Provider>
  );
}

// 投稿一覧コンポーネント（楽観的更新付き）
export function PostList() {
  const utils = trpc.useUtils();

  // 型安全なクエリ（IntelliSenseが効く）
  const { data, isLoading } = trpc.posts.list.useQuery({ limit: 20 });

  const createPost = trpc.posts.create.useMutation({
    onSuccess: () => {
      utils.posts.list.invalidate(); // キャッシュ無効化
    },
  });

  const deletePost = trpc.posts.delete.useMutation({
    // 楽観的更新
    onMutate: async ({ id }) => {
      await utils.posts.list.cancel();
      const previous = utils.posts.list.getData();
      utils.posts.list.setData({ limit: 20 }, old => ({
        ...old!,
        items: old!.items.filter(p => p.id !== id),
      }));
      return { previous };
    },
    onError: (_, __, ctx) => {
      if (ctx?.previous) utils.posts.list.setData({ limit: 20 }, ctx.previous);
    },
  });

  if (isLoading) return <Spinner />;

  return (
    <ul>
      {data?.items.map(post => (
        <li key={post.id}>
          {post.title} by {post.author.name}
          <button onClick={() => deletePost.mutate({ id: post.id })}>削除</button>
        </li>
      ))}
    </ul>
  );
}
```

---

## まとめ

Claude CodetRPC + Zod + PrismaでAPI設計する：

1. **CLAUDE.md** にpublicProcedure/protectedProcedureの2段階・Zodバリデーション・Prisma Select型を明記
2. **postSelect satisfies Prisma.PostSelect** でコンパイル時に型安全なフィールド選択
3. **所有権チェック** をupdate/deleteプロシージャで統一パターン化
4. **楽観的更新** でonMutate/onError/setDataパターンを標準化

---

*型設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
