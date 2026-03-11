---
title: "Claude CodeでgRPCを設計する：Proto定義・型安全なサービス間通信・ストリーミング"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "grpc", "microservices"]
published: true
---

## はじめに

マイクロサービス間の通信にREST/JSONを使うと、スキーマが曖昧になり型安全性が失われる。gRPCでProtobuf定義から型安全なサービス間通信を実現する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにgRPC設計ルールを書く

```markdown
## gRPC設計ルール

### Proto定義
- .protoファイルはmonorepoの shared/proto/ に集中管理
- フィールド番号は変更禁止（後方互換性）
- フィールド削除はreservedで予約（番号の再利用防止）
- パッケージ名はドメインを反映（例: order.v1）

### サービス設計
- 1サービス = 1ドメイン（OrderService、UserServiceなど）
- Unary RPC: 通常のリクエスト/レスポンス
- Server Streaming: リアルタイム更新（進捗通知など）
- Bidirectional Streaming: チャット・ライブコラボなど

### エラーハンドリング
- gRPCステータスコード（NOT_FOUND, ALREADY_EXISTS, PERMISSION_DENIED等）を使う
- RESTに変換する場合はgrpc-gatewayを使う
- Metadata でトレースIDを伝播
```

---

## gRPCサービスの生成

```
注文管理のgRPCサービスを設計してください。

要件：
- Proto定義（OrderService）
- Node.js/TypeScript実装
- Unary RPC + Server Streaming
- エラーハンドリング（gRPCステータスコード）
- gRPC-Web（ブラウザクライアント対応）

生成ファイル: proto/order.proto, src/grpc/orderServer.ts
```

---

## 生成されるgRPC実装

```protobuf
// proto/order/v1/order.proto
syntax = "proto3";

package order.v1;

import "google/protobuf/timestamp.proto";

service OrderService {
  // Unary: 注文作成
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  // Unary: 注文取得
  rpc GetOrder(GetOrderRequest) returns (Order);
  // Server Streaming: 注文状態のリアルタイム更新
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusUpdate);
  // Unary: 注文キャンセル
  rpc CancelOrder(CancelOrderRequest) returns (Order);
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  int64 price_cents = 3;
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  int64 total_cents = 5;
  google.protobuf.Timestamp created_at = 6;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}

message GetOrderRequest {
  string order_id = 1;
}

message WatchOrderStatusRequest {
  string order_id = 1;
}

message OrderStatusUpdate {
  string order_id = 1;
  OrderStatus status = 2;
  string message = 3;
  google.protobuf.Timestamp updated_at = 4;
}

message CancelOrderRequest {
  string order_id = 1;
  string reason = 2;
}
```

```typescript
// src/grpc/orderServer.ts
import * as grpc from '@grpc/grpc-js';
import { OrderServiceImplementation } from '@proto/order/v1/order_grpc_pb';
import { Status } from '@grpc/grpc-js/build/src/constants';

const orderServiceImpl: OrderServiceImplementation = {
  async createOrder(call, callback) {
    const { userId, items } = call.request.toObject();
    const traceId = call.metadata.get('x-trace-id')[0] as string;

    try {
      // 在庫確認
      const inventoryOk = await checkInventory(items);
      if (!inventoryOk) {
        return callback({
          code: Status.FAILED_PRECONDITION,
          message: 'Insufficient inventory',
        });
      }

      const order = await prisma.order.create({
        data: {
          userId,
          items: { create: items },
          status: 'PENDING',
          totalCents: items.reduce((sum, item) => sum + item.priceCents * item.quantity, 0),
        },
        include: { items: true },
      });

      const response = new Order();
      response.setId(order.id);
      response.setUserId(order.userId);
      response.setStatus(OrderStatus.ORDER_STATUS_PENDING);
      response.setTotalCents(order.totalCents);

      callback(null, response);
    } catch (err) {
      logger.error({ err, traceId }, 'CreateOrder failed');
      callback({
        code: Status.INTERNAL,
        message: 'Internal server error',
      });
    }
  },

  async getOrder(call, callback) {
    const { orderId } = call.request.toObject();

    const order = await prisma.order.findUnique({
      where: { id: orderId },
      include: { items: true },
    });

    if (!order) {
      return callback({
        code: Status.NOT_FOUND,
        message: `Order ${orderId} not found`,
      });
    }

    callback(null, toProtoOrder(order));
  },

  // Server Streaming: 注文ステータスをリアルタイムで送信
  watchOrderStatus(call) {
    const { orderId } = call.request.toObject();

    // Redisのサブスクライブで状態変化を検知
    const subscriber = redis.duplicate();
    subscriber.subscribe(`order:${orderId}:status`, (err) => {
      if (err) {
        call.destroy(err);
        return;
      }
    });

    subscriber.on('message', (_channel, message) => {
      const update = JSON.parse(message);
      const statusUpdate = new OrderStatusUpdate();
      statusUpdate.setOrderId(orderId);
      statusUpdate.setStatus(update.status);
      statusUpdate.setMessage(update.message);

      call.write(statusUpdate);

      // 終端状態（delivered/cancelled）ではストリームを終了
      if (['DELIVERED', 'CANCELLED'].includes(update.status)) {
        subscriber.unsubscribe();
        call.end();
      }
    });

    call.on('cancelled', () => {
      subscriber.unsubscribe();
    });
  },
};

export function startGrpcServer(): void {
  const server = new grpc.Server();
  server.addService(OrderServiceService, orderServiceImpl);

  const PORT = process.env.GRPC_PORT ?? '50051';
  server.bindAsync(
    `0.0.0.0:${PORT}`,
    grpc.ServerCredentials.createInsecure(), // 本番はmTLSを使う
    (err, port) => {
      if (err) throw err;
      server.start();
      logger.info({ port }, 'gRPC server started');
    }
  );
}
```

---

## クライアント実装

```typescript
// src/grpc/orderClient.ts
import * as grpc from '@grpc/grpc-js';
import { OrderServiceClient } from '@proto/order/v1/order_grpc_pb';

const client = new OrderServiceClient(
  process.env.ORDER_SERVICE_GRPC_URL ?? 'localhost:50051',
  grpc.credentials.createInsecure()
);

// Promisify Unary RPC
function createOrderGrpc(userId: string, items: OrderItem[]): Promise<Order> {
  return new Promise((resolve, reject) => {
    const req = new CreateOrderRequest();
    req.setUserId(userId);
    req.setItemsList(items.map(toProtoItem));

    // traceIdをメタデータで伝播
    const metadata = new grpc.Metadata();
    metadata.set('x-trace-id', getCurrentTraceId() ?? '');

    client.createOrder(req, metadata, (err, response) => {
      if (err) reject(err);
      else resolve(response!);
    });
  });
}

// Server Streaming
function watchOrderStatus(orderId: string, onUpdate: (update: OrderStatusUpdate) => void): void {
  const req = new WatchOrderStatusRequest();
  req.setOrderId(orderId);

  const stream = client.watchOrderStatus(req);

  stream.on('data', (update: OrderStatusUpdate) => {
    onUpdate(update);
  });

  stream.on('error', (err) => {
    logger.error({ err, orderId }, 'Order status stream error');
  });

  stream.on('end', () => {
    logger.info({ orderId }, 'Order status stream ended');
  });
}
```

---

## まとめ

Claude CodeでgRPCを設計する：

1. **CLAUDE.md** にProto一元管理・フィールド番号変更禁止・gRPCステータスコードを明記
2. **Protobuf** で型安全なスキーマ定義（REST/JSONより強力な型システム）
3. **Server Streaming** でリアルタイム状態更新（WebSocketの代替としても有効）
4. **Metadata** でtraceIdを伝播（マイクロサービス間で分散トレーシング）

---

*gRPC設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
