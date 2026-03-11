---
title: "Claude CodeでGraphQL APIを設計する：スキーマファーストとN+1問題の解決"
emoji: "🔷"
type: "tech"
topics: ["claudecode", "graphql", "nodejs", "typescript", "api設計"]
published: true
---

## はじめに

GraphQL APIを設計する際、スキーマ設計・リゾルバ実装・N+1問題・認証の4つが複雑に絡み合う。Claude Codeにスキーマファーストで設計させると、ベストプラクティスに沿ったAPIが得られる。

---

## CLAUDE.mdにGraphQLルールを書く

```markdown
## GraphQLルール

### 設計方針
- スキーマファースト: schema.graphql を先に定義してからリゾルバを実装
- コードジェネレーター: GraphQL Code Generator でTypeScript型を自動生成
- スキーマファイル: src/graphql/schema/ に機能ごとに分割

### N+1問題（必須対策）
- リゾルバでDBを直接叩かない
- 全リレーションにDataLoaderを使う
- クエリ深度制限: 最大5レベル
- クエリ複雑度制限: 最大100

### 認証・認可
- 全クエリ: 認証ミドルウェアでJWT検証
- フィールドレベル認可: @auth ディレクティブを使う
- 管理者専用フィールドには @adminOnly ディレクティブ

### エラーハンドリング
- GraphQLErrorを使う（HTTPステータスをerrorのextensionsに含める）
- ユーザー向けエラーメッセージとデバッグ情報を分離
- センシティブな情報をエラーに含めない

### 禁止
- リゾルバ内で直接SQL/Prismaを実行（サービス層を通す）
- __resolveType の乱用（Union型は慎重に使う）
```

---

## GraphQLスキーマの生成

```
GraphQL スキーマを設計してください（スキーマファースト）。

対象ドメイン: ブログプラットフォーム
- User: id, email, name, posts
- Post: id, title, content, author, comments, publishedAt
- Comment: id, content, author, post

要件：
- ページネーション: Relay Connection Spec（cursor-based）
- 変更操作: createPost, updatePost, deletePost, addComment
- 認証: @auth ディレクティブ（未認証ユーザーはデータを見られない）
- エラー型: UserError（ユーザー向け）/ InternalError（サーバー側）

出力: src/graphql/schema/*.graphql
```

---

## 生成されるスキーマの例

```graphql
# src/graphql/schema/user.graphql
type User {
  id: ID!
  email: String! @auth(requires: SELF_OR_ADMIN)
  name: String!
  posts(first: Int, after: String): PostConnection!
  createdAt: DateTime!
}

# src/graphql/schema/post.graphql
type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments(first: Int, after: String): CommentConnection!
  publishedAt: DateTime
  isPublished: Boolean!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type PostEdge {
  node: Post!
  cursor: String!
}

# src/graphql/schema/mutation.graphql
type Mutation {
  createPost(input: CreatePostInput!): CreatePostPayload! @auth
  updatePost(id: ID!, input: UpdatePostInput!): UpdatePostPayload! @auth
  deletePost(id: ID!): DeletePostPayload! @auth
  addComment(postId: ID!, content: String!): AddCommentPayload! @auth
}

type CreatePostPayload {
  post: Post
  userErrors: [UserError!]!
}
```

---

## DataLoaderでN+1問題を解決する

```
以下のリゾルバのN+1問題をDataLoaderで解決してください。

問題のあるリゾルバ:
- Post.author: 各投稿ごとにUserをDBから取得している
- Post.comments: 各投稿ごとにCommentをDBから取得している

要件：
- DataLoaderをリクエストスコープで生成（グローバルキャッシュ禁止）
- バッチ関数はPrisma findMany で IN句を使う
- TypeScript型安全

生成ファイル:
- src/graphql/dataloaders/userLoader.ts
- src/graphql/dataloaders/commentLoader.ts
- src/graphql/context.ts（DataLoaderをコンテキストに格納）
```

```typescript
// src/graphql/dataloaders/userLoader.ts
import DataLoader from 'dataloader';
import { prisma } from '../../lib/prisma';

export function createUserLoader() {
  return new DataLoader<string, User>(async (userIds) => {
    const users = await prisma.user.findMany({
      where: { id: { in: [...userIds] } },
    });
    const userMap = new Map(users.map(u => [u.id, u]));
    return userIds.map(id => userMap.get(id) ?? new Error(`User not found: ${id}`));
  });
}

// src/graphql/context.ts
export function createContext({ req }: { req: Request }) {
  return {
    user: verifyToken(req.headers.authorization),
    loaders: {
      user: createUserLoader(),      // リクエストごとに新規作成
      comment: createCommentLoader(),
    },
  };
}
```

---

## クエリ複雑度制限の設定

```
GraphQL クエリの深度と複雑度を制限してください。

要件：
- 深度制限: 最大5レベル
- 複雑度制限: 最大100（フィールドごとにコスト設定可能）
- limitationはpre-executionフックで実施
- 制限超過時のエラーメッセージを明確にする

ライブラリ: graphql-depth-limit + graphql-validation-complexity
```

---

## まとめ

Claude CodeでGraphQL APIを設計する：

1. **CLAUDE.md** にスキーマファースト・DataLoader必須・深度制限を明記
2. **スキーマを先に生成** させてからリゾルバを実装
3. **DataLoader** でN+1問題を防止
4. **@auth ディレクティブ** でフィールドレベル認可を統一
5. **複雑度制限** でDDoS攻撃を防ぐ

---

*GraphQLのN+1・認証漏れ・スキーマ設計問題を自動検出するスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
