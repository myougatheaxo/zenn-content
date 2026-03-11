---
title: "Claude CodeでGraphQL Subscriptionsを設計する：リアルタイム更新・Redis Pub/Sub・スケーリング"
emoji: "📡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "graphql", "redis"]
published: true
published_at: "2026-03-13 11:00"
---

## はじめに

GraphQL SubscriptionsでWebSocketベースのリアルタイム更新を実装する——通知・チャット・ライブダッシュボードをRedis Pub/SubとApollo Serverで水平スケーリング可能な構成にClaude Codeに設計させる。

---

## CLAUDE.mdにGraphQL Subscriptions設計ルールを書く

```markdown
## GraphQL Subscriptions設計ルール

### インフラ
- WebSocketは graphql-ws（ws-link）プロトコル
- マルチサーバー: Redis Pub/Subでサーバー間イベント共有
- 認証: Connection initでJWT検証

### パフォーマンス
- 1接続1テナントの原則（接続ごとに認証情報保持）
- Filterはサーバーサイドで実装（クライアント送信量削減）
- メッセージサイズ: 最大50KB（大きなオブジェクトは変更差分のみ送信）

### スケーリング
- Redis Pub/Sub Channelはエンティティごと（orders:userId:xxx）
- 不活性接続のタイムアウト: 30分でKeepalive確認→切断
- 最大接続数: サーバー1台あたり5,000接続
```

---

## GraphQL Subscriptions実装の生成

```
GraphQL Subscriptionsのリアルタイムシステムを設計してください。

要件：
- Apollo Server + graphql-ws
- Redis Pub/Sub（マルチサーバー対応）
- 認証（JWT検証）
- 注文状態・メッセージのリアルタイム更新

生成ファイル: src/graphql/subscriptions/
```

---

## 生成されるGraphQL Subscriptions実装

```typescript
// src/graphql/subscriptions/pubsub.ts
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

// Redis Pub/Sub（マルチサーバー対応）
export const pubsub = new RedisPubSub({
  publisher: new Redis(process.env.REDIS_URL!),
  subscriber: new Redis(process.env.REDIS_URL!), // subscribeは専用接続
});

// イベントチャンネル定数
export const CHANNELS = {
  ORDER_UPDATED:    (userId: string)    => `ORDER_UPDATED:${userId}`,
  MESSAGE_CREATED:  (conversationId: string) => `MESSAGE_CREATED:${conversationId}`,
  NOTIFICATION:     (userId: string)    => `NOTIFICATION:${userId}`,
} as const;
```

```typescript
// src/graphql/schema/subscriptions.ts

export const subscriptionTypeDefs = gql`
  type Subscription {
    # 注文状態変更（認証済みユーザーの自分の注文のみ）
    orderUpdated(orderId: ID!): Order!

    # 新着メッセージ（会話参加者のみ）
    messageCreated(conversationId: ID!): Message!

    # リアルタイム通知
    notificationReceived: Notification!
  }
`;

export const subscriptionResolvers = {
  Subscription: {
    orderUpdated: {
      subscribe: withFilter(
        // Redux Pub/Subチャンネルを購読
        (_root, { orderId }, { userId }) => {
          // ユーザーのチャンネルを購読
          return pubsub.asyncIterator(CHANNELS.ORDER_UPDATED(userId));
        },
        // フィルタリング: 指定されたorderIdのイベントのみ通過
        (payload, { orderId }) => payload.orderUpdated.id === orderId
      ),
      resolve: (payload) => payload.orderUpdated,
    },

    messageCreated: {
      subscribe: withFilter(
        async (_root, { conversationId }, { userId }) => {
          // 会話の参加者チェック
          const member = await prisma.conversationMember.findFirst({
            where: { conversationId, userId },
          });
          if (!member) throw new ForbiddenError('Not a member of this conversation');

          return pubsub.asyncIterator(CHANNELS.MESSAGE_CREATED(conversationId));
        },
        (payload, { conversationId }) =>
          payload.messageCreated.conversationId === conversationId
      ),
      resolve: (payload) => payload.messageCreated,
    },

    notificationReceived: {
      subscribe: (_root, _args, { userId }) =>
        pubsub.asyncIterator(CHANNELS.NOTIFICATION(userId)),
      resolve: (payload) => payload.notification,
    },
  },
};
```

```typescript
// src/graphql/server.ts — Apollo Server with WebSocket

import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';

const schema = makeExecutableSchema({ typeDefs, resolvers });

// WebSocketサーバー（Subscriptions専用）
const wsServer = new WebSocketServer({ server: httpServer, path: '/graphql' });

const wsServerCleanup = useServer(
  {
    schema,
    // 接続時の認証（JWT検証）
    onConnect: async (ctx) => {
      const token = ctx.connectionParams?.authorization as string;
      if (!token) throw new Error('Authentication required');

      const payload = await verifyJWT(token.replace('Bearer ', ''));
      if (!payload) throw new Error('Invalid or expired token');

      // コンテキストにユーザー情報を保存
      return { userId: payload.sub, role: payload.role };
    },
    // 接続切断時のクリーンアップ
    onDisconnect: (ctx) => {
      logger.info({ userId: ctx.extra?.userId }, 'WebSocket disconnected');
    },
    // リクエストごとのコンテキスト（resolverに渡る）
    context: (ctx) => ({
      userId: (ctx.extra as any)?.userId,
      role: (ctx.extra as any)?.role,
    }),
  },
  wsServer
);

// Apollo Server（HTTP QueryとMutation）
const server = new ApolloServer({
  schema,
  plugins: [{
    async serverWillStart() {
      return {
        async drainServer() {
          await wsServerCleanup.dispose();
        },
      };
    },
  }],
});
```

```typescript
// src/services/orderService.ts — イベント発行側

export async function updateOrderStatus(
  orderId: string,
  newStatus: OrderStatus
): Promise<Order> {
  const order = await prisma.order.update({
    where: { id: orderId },
    data: { status: newStatus, updatedAt: new Date() },
  });

  // Redis Pub/Subで全サーバーのWebSocket接続に配信
  await pubsub.publish(CHANNELS.ORDER_UPDATED(order.userId), {
    orderUpdated: order,
  });

  return order;
}

// メッセージ送信
export async function sendMessage(params: SendMessageParams): Promise<Message> {
  const message = await prisma.message.create({
    data: {
      conversationId: params.conversationId,
      senderId: params.userId,
      content: params.content,
    },
  });

  // 会話参加者全員に通知
  await pubsub.publish(CHANNELS.MESSAGE_CREATED(params.conversationId), {
    messageCreated: message,
  });

  return message;
}
```

---

## フロントエンド（React + Apollo Client）

```typescript
// src/components/OrderStatus.tsx
import { gql, useSubscription } from '@apollo/client';

const ORDER_UPDATED = gql`
  subscription OrderUpdated($orderId: ID!) {
    orderUpdated(orderId: $orderId) {
      id
      status
      updatedAt
    }
  }
`;

export function OrderStatusDisplay({ orderId }: { orderId: string }) {
  const { data, loading } = useSubscription(ORDER_UPDATED, {
    variables: { orderId },
  });

  if (loading) return <span>Connecting...</span>;

  return (
    <span className={`status-${data?.orderUpdated.status}`}>
      {data?.orderUpdated.status}
    </span>
  );
}
```

---

## まとめ

Claude CodeでGraphQL Subscriptionsを設計する：

1. **CLAUDE.md** にRedis Pub/Sub必須・接続時JWT検証・withFilterでサーバーサイドフィルタを明記
2. **Redis Pub/Sub** でマルチサーバー対応——どのサーバーに繋がっても正しいイベントを受信
3. **withFilter** で不要なイベントを送信前にフィルタリング（ネットワーク帯域節約）
4. **onConnect** でWebSocket接続時にJWT検証し、以降のリクエストは再検証不要

---

*GraphQL Subscriptions設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
