---
title: "Claude CodeでGraphQL Federationを設計する：サブグラフ分割・@key・ゲートウェイ統合"
emoji: "🌐"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "graphql", "microservices"]
published: true
published_at: "2026-03-13 06:00"
---

## はじめに

マイクロサービスごとにGraphQLスキーマを持たせ、統合ゲートウェイで1つのAPIに見せる——Apollo Federationの@key・@external・@requiresを使ったサブグラフ設計をClaude Codeに生成させる。

---

## CLAUDE.mdにGraphQL Federation設計ルールを書く

```markdown
## GraphQL Federation設計ルール

### サブグラフ分割
- 1マイクロサービス = 1サブグラフ（責務に従って分割）
- エンティティは@keyで定義（DB主キーが基本）
- 他サービスのフィールドを参照: @external + @requires

### エンティティ設計
- @key(fields: "id") で主キーを公開
- 他サブグラフへの参照は stub型 + @extends
- N+1問題: Dataloaderで__resolveReference をバッチ処理

### ゲートウェイ
- Apollo Router（Rust実装）を使用（高性能）
- Persisted Queries: セキュリティ + CDNキャッシュ
- レート制限: depth 10, aliasCount 10
```

---

## GraphQL Federationの生成

```
GraphQL Federationマイクロサービスアーキテクチャを設計してください。

要件：
- ユーザーサービス + 商品サービス + 注文サービス
- @keyによるエンティティ連携
- Dataloaderバッチ処理
- Apollo Routerゲートウェイ
- 認証伝播

生成ファイル: services/users/src/graphql/, services/products/src/graphql/
```

---

## 生成されるGraphQL Federation実装

```typescript
// services/users/src/graphql/schema.ts — ユーザーサブグラフ

import { buildSubgraphSchema } from '@apollo/subgraph';
import { gql } from 'graphql-tag';

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.3", import: ["@key", "@shareable"])

  type User @key(fields: "id") {
    id: ID!
    email: String!
    name: String!
    createdAt: String!
    # ordersはOrderサービスが追加するため、ここには含めない
  }

  type Query {
    user(id: ID!): User
    me: User
  }

  type Mutation {
    updateProfile(name: String!): User!
  }
`;

const resolvers = {
  Query: {
    user: async (_: unknown, { id }: { id: string }, { prisma }: Context) =>
      prisma.user.findUnique({ where: { id } }),
    me: async (_: unknown, __: unknown, { user, prisma }: Context) => {
      if (!user) throw new AuthenticationError('Not authenticated');
      return prisma.user.findUnique({ where: { id: user.id } });
    },
  },
  Mutation: {
    updateProfile: async (_: unknown, { name }: { name: string }, { user, prisma }: Context) => {
      if (!user) throw new AuthenticationError('Not authenticated');
      return prisma.user.update({ where: { id: user.id }, data: { name } });
    },
  },
  User: {
    // __resolveReferenceでFederation経由のエンティティ解決
    __resolveReference: async (reference: { id: string }, { prisma }: Context) =>
      prisma.user.findUnique({ where: { id: reference.id } }),
  },
};

export const userSubgraphSchema = buildSubgraphSchema({ typeDefs, resolvers });
```

```typescript
// services/products/src/graphql/schema.ts — 商品サブグラフ

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.3", import: ["@key", "@external", "@requires"])

  type Product @key(fields: "id") {
    id: ID!
    name: String!
    price: Float!
    description: String
    inventory: Int!
    category: String!
  }

  type Query {
    product(id: ID!): Product
    products(category: String, page: Int, size: Int): ProductConnection!
  }

  type ProductConnection {
    items: [Product!]!
    totalCount: Int!
    hasNextPage: Boolean!
  }
`;

const resolvers = {
  Query: {
    product: async (_: unknown, { id }: { id: string }, { loaders }: Context) =>
      loaders.product.load(id), // Dataloaderバッチ

    products: async (_: unknown, args: ProductsArgs, { prisma }: Context) => {
      const { category, page = 1, size = 20 } = args;
      const where = category ? { category } : {};

      const [items, totalCount] = await prisma.$transaction([
        prisma.product.findMany({ where, take: size, skip: (page - 1) * size }),
        prisma.product.count({ where }),
      ]);

      return { items, totalCount, hasNextPage: page * size < totalCount };
    },
  },
  Product: {
    __resolveReference: async (reference: { id: string }, { loaders }: Context) =>
      loaders.product.load(reference.id),
  },
};
```

```typescript
// services/orders/src/graphql/schema.ts — 注文サブグラフ（User・Productを参照）

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.3",
      import: ["@key", "@external", "@requires", "@provides"])

  # Userエンティティを参照（@extends）
  type User @key(fields: "id") {
    id: ID! @external
    orders: [Order!]!  # Userに注文リストを追加
  }

  # Productエンティティを参照
  type Product @key(fields: "id") {
    id: ID! @external
    name: String! @external
  }

  type Order @key(fields: "id") {
    id: ID!
    status: OrderStatus!
    totalAmount: Float!
    createdAt: String!
    user: User!
    items: [OrderItem!]!
  }

  type OrderItem {
    product: Product!
    quantity: Int!
    unitPrice: Float!
  }

  enum OrderStatus {
    PENDING
    CONFIRMED
    SHIPPED
    DELIVERED
    CANCELLED
  }

  type Mutation {
    createOrder(items: [OrderItemInput!]!): Order!
    cancelOrder(id: ID!): Order!
  }
`;

const resolvers = {
  User: {
    // UserサービスのUserにordersフィールドを追加
    orders: async (user: { id: string }, _: unknown, { prisma }: Context) =>
      prisma.order.findMany({
        where: { userId: user.id },
        orderBy: { createdAt: 'desc' },
        take: 10,
      }),
  },
  Order: {
    user: (order: { userId: string }) => ({ __typename: 'User', id: order.userId }),
    items: async (order: { id: string }, _: unknown, { prisma }: Context) =>
      prisma.orderItem.findMany({
        where: { orderId: order.id },
        include: { product: true },
      }),
    __resolveReference: async (ref: { id: string }, { prisma }: Context) =>
      prisma.order.findUnique({ where: { id: ref.id } }),
  },
  OrderItem: {
    product: (item: { productId: string }) => ({ __typename: 'Product', id: item.productId }),
  },
  Mutation: {
    createOrder: async (_: unknown, { items }: CreateOrderArgs, { user, prisma }: Context) => {
      if (!user) throw new AuthenticationError('Not authenticated');

      // 価格をProduct serviceから取得して合計計算
      const productIds = items.map(i => i.productId);
      const products = await fetchProductPrices(productIds); // gRPC/HTTP call

      const totalAmount = items.reduce((sum, item) => {
        const product = products.find(p => p.id === item.productId)!;
        return sum + product.price * item.quantity;
      }, 0);

      return prisma.order.create({
        data: {
          userId: user.id,
          status: 'PENDING',
          totalAmount,
          items: {
            create: items.map(item => ({
              productId: item.productId,
              quantity: item.quantity,
              unitPrice: products.find(p => p.id === item.productId)!.price,
            })),
          },
        },
      });
    },
  },
};
```

```yaml
# router.yaml — Apollo Router設定

federation_version: =2.3.0

supergraph:
  listen: 0.0.0.0:4000
  introspection: false  # 本番はオフ

subgraphs:
  users:
    routing_url: http://users-service:4001/graphql
  products:
    routing_url: http://products-service:4002/graphql
  orders:
    routing_url: http://orders-service:4003/graphql

authentication:
  router:
    jwt:
      jwks:
        - url: https://auth.example.com/.well-known/jwks.json

authorization:
  preview_directives:
    enabled: true

limits:
  max_depth: 10
  max_aliases: 10
  max_root_fields: 20

traffic_shaping:
  router:
    timeout: 30s
  all:
    timeout: 15s
    deduplicate_query: true  # 同一クエリの重複実行防止
```

---

## まとめ

Claude CodeでGraphQL Federationを設計する：

1. **CLAUDE.md** に1サービス=1サブグラフ・@key主キー公開・Dataloaderバッチ・Apollo Routerを明記
2. **@key(fields: "id")** でエンティティを識別し、他サービスからstub型で参照（Routerが自動解決）
3. **__resolveReference** でDataloaderバッチを組み合わせ、N+1クエリをサブグラフ間でも防止
4. **User.orders** のように別サービスの型にフィールドを追加できるのがFederationの核心

---

*GraphQL設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
