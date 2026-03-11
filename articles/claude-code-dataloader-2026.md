---
title: "Claude CodeでDataLoaderを設計する：GraphQL N+1問題をバッチで解決"
emoji: "🔷"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "graphql", "prisma"]
published: true
---

## はじめに

GraphQLでN+1問題を放置すると、100件のPostを返すクエリでDBに101回クエリが飛ぶ。DataLoaderでバッチ処理に変換して1回にまとめる。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにDataLoader設計ルールを書く

```markdown
## DataLoader設計ルール

### 必須ルール
- GraphQLリゾルバーからの直接DBアクセス禁止（DataLoader経由のみ）
- DataLoaderは1リクエスト = 1インスタンス（コンテキストに注入）
- 最大バッチサイズ: 100件（DB IN句の上限）

### キャッシュ戦略
- DataLoaderのデフォルトキャッシュはリクエストスコープ
- 変更操作後はloader.clearAll()でキャッシュクリア
- セキュリティ: 別ユーザーのデータはキャッシュしない

### バッチ関数の実装
- 入力IDの順序通りに結果を返す（DataLoaderの要件）
- 存在しないIDにはnullを返す（エラーにしない）
- 関連エンティティは groupBy / Map で効率的にマッピング
```

---

## DataLoaderシステムの生成

```
GraphQL向けDataLoaderシステムを設計してください。

エンティティ: User, Post, Comment
要件：
- userLoader（IDバッチ）
- postsByAuthorLoader（authorIdバッチ）
- commentCountLoader（postIdバッチ）
- 変更後のキャッシュクリア

生成ファイル: src/graphql/dataLoaders.ts
```

---

## 生成されるDataLoader実装

```typescript
// src/graphql/dataLoaders.ts
import DataLoader from 'dataloader';

export function createDataLoaders() {
  // ==============================
  // 基本: IDバッチローダー
  // ==============================
  const userLoader = new DataLoader<string, User | null>(
    async (ids: readonly string[]) => {
      const users = await prisma.user.findMany({
        where: { id: { in: [...ids] } },
      });

      // IDの順序を保持（DataLoaderの要件：入力と出力の長さ・順序が一致）
      const userMap = new Map(users.map(u => [u.id, u]));
      return ids.map(id => userMap.get(id) ?? null);
    },
    {
      maxBatchSize: 100,
      cache: true, // リクエスト内でキャッシュ（デフォルトtrue）
    }
  );

  // ==============================
  // 1対多: authorIdから投稿リスト
  // ==============================
  const postsByAuthorLoader = new DataLoader<string, Post[]>(
    async (authorIds: readonly string[]) => {
      const posts = await prisma.post.findMany({
        where: { authorId: { in: [...authorIds] } },
        orderBy: { createdAt: 'desc' },
      });

      // authorIdでグループ化
      const postsByAuthor = new Map<string, Post[]>();
      for (const authorId of authorIds) {
        postsByAuthor.set(authorId, []); // 投稿ゼロのユーザーも含める
      }
      for (const post of posts) {
        postsByAuthor.get(post.authorId)?.push(post);
      }

      return authorIds.map(id => postsByAuthor.get(id) ?? []);
    }
  );

  // ==============================
  // 集計: postIdごとのコメント数
  // ==============================
  const commentCountLoader = new DataLoader<string, number>(
    async (postIds: readonly string[]) => {
      const counts = await prisma.comment.groupBy({
        by: ['postId'],
        where: { postId: { in: [...postIds] } },
        _count: { id: true },
      });

      const countMap = new Map(counts.map(c => [c.postId, c._count.id]));
      return postIds.map(id => countMap.get(id) ?? 0);
    }
  );

  // ==============================
  // 関連: タグの一括取得（多対多）
  // ==============================
  const tagsByPostLoader = new DataLoader<string, Tag[]>(
    async (postIds: readonly string[]) => {
      const postTags = await prisma.postTag.findMany({
        where: { postId: { in: [...postIds] } },
        include: { tag: true },
      });

      const tagsByPost = new Map<string, Tag[]>();
      for (const postId of postIds) {
        tagsByPost.set(postId, []);
      }
      for (const pt of postTags) {
        tagsByPost.get(pt.postId)?.push(pt.tag);
      }

      return postIds.map(id => tagsByPost.get(id) ?? []);
    }
  );

  return { userLoader, postsByAuthorLoader, commentCountLoader, tagsByPostLoader };
}

export type DataLoaders = ReturnType<typeof createDataLoaders>;
```

---

## コンテキストに注入してリゾルバーで使用

```typescript
// src/graphql/context.ts
export interface Context {
  user: AuthUser | null;
  loaders: DataLoaders;
}

export function createContext({ req }: { req: Request }): Context {
  return {
    user: req.user ?? null,
    loaders: createDataLoaders(), // リクエストごとに新規作成（キャッシュをスコープ限定）
  };
}

// src/graphql/resolvers.ts
export const resolvers = {
  Post: {
    // NG: 個別クエリ（N+1問題）
    // author: (post) => prisma.user.findUnique({ where: { id: post.authorId } }),

    // OK: DataLoader経由でバッチ処理
    author: (post: Post, _: unknown, ctx: Context) => {
      return ctx.loaders.userLoader.load(post.authorId);
    },

    commentCount: (post: Post, _: unknown, ctx: Context) => {
      return ctx.loaders.commentCountLoader.load(post.id);
    },

    tags: (post: Post, _: unknown, ctx: Context) => {
      return ctx.loaders.tagsByPostLoader.load(post.id);
    },
  },

  User: {
    posts: (user: User, _: unknown, ctx: Context) => {
      return ctx.loaders.postsByAuthorLoader.load(user.id);
    },
  },
};
```

---

## 変更後のキャッシュクリア

```typescript
// Mutation後にキャッシュを無効化
const resolvers = {
  Mutation: {
    updateUser: async (_: unknown, { id, input }: UpdateUserArgs, ctx: Context) => {
      const user = await prisma.user.update({
        where: { id },
        data: input,
      });

      // 更新後はキャッシュをクリア（古いデータを返さないため）
      ctx.loaders.userLoader.clear(id);

      return user;
    },

    deletePost: async (_: unknown, { id }: { id: string }, ctx: Context) => {
      await prisma.post.delete({ where: { id } });

      // 関連するすべてのローダーをクリア
      ctx.loaders.commentCountLoader.clear(id);
      ctx.loaders.tagsByPostLoader.clear(id);

      return true;
    },
  },
};
```

---

## まとめ

Claude CodeでDataLoaderを設計する：

1. **CLAUDE.md** にリゾルバー直接DBアクセス禁止・リクエストスコープ・バッチサイズ上限を明記
2. **IDの順序保持** でDataLoaderの正確なマッピングを保証（Mapを使う）
3. **1対多はグループ化** で関連エンティティを効率的に取得
4. **変更後はclear()** でキャッシュを明示的に無効化

---

*DataLoader/GraphQL設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
