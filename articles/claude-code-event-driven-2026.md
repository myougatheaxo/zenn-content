---
title: "Claude Codeでイベント駆動アーキテクチャを設計する：疎結合なサービス通信"
emoji: "📡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "アーキテクチャ", "マイクロサービス"]
published: true
---

## はじめに

サービス間の直接呼び出しが増えると密結合になり、変更のたびに連鎖的な修正が発生する。イベント駆動アーキテクチャでサービスを疎結合にする設計をClaude Codeに任せる。

---

## CLAUDE.mdにイベント駆動アーキテクチャルールを書く

```markdown
## イベント駆動アーキテクチャルール

### 基本原則
- サービス間の直接HTTP呼び出しは禁止（同期通信が必要な場合のみ許可）
- 全ての状態変更はイベントとして発行する
- イベントは不変（過去のイベントは変更しない）

### イベント設計
- 命名規則: {ドメイン}.{エンティティ}.{動詞の過去形}
  例: order.item.added, user.account.created, payment.completed
- ペイロードには必ず: eventId, timestamp, version, data
- 後方互換性: イベントスキーマを変更する場合はバージョンを上げる

### 実装
- Node.js単一サービス内: EventEmitter（軽量）
- 複数サービス間: Redis Pub/Sub または BullMQ Events
- スキーマバリデーション: Zodでペイロードを検証

### 信頼性
- イベントはDB保存してから発行（Outboxパターン）
- 消費者は冪等性を保証（同じイベントが2回来ても問題ない）
- Dead Letter Queue: 3回失敗したイベントはDLQへ
```

---

## イベントエミッターの型安全な設計

```
型安全なイベントエミッターシステムを生成してください。

要件：
- TypeScript: イベント名とペイロード型の対応を型で保証
- 全イベントにeventId・timestamp・versionを自動付与
- エラーハンドリング: リスナーがエラーを投げても他のリスナーを止めない
- デバッグ: 全イベントをログに記録

イベント定義:
- user.created: { userId: string, email: string, name: string }
- order.placed: { orderId: string, userId: string, items: Array<{productId: string, quantity: number}>, total: number }
- payment.completed: { paymentId: string, orderId: string, amount: number }

生成ファイル:
- src/events/types.ts（イベント型定義）
- src/events/eventBus.ts（型安全なEventEmitter）
- src/events/handlers/（各ドメインのハンドラー）
```

---

## 生成される型安全なEventBus

```typescript
// src/events/types.ts
export interface EventPayload {
  eventId: string;
  timestamp: string;
  version: number;
}

export interface DomainEvents {
  'user.created': EventPayload & {
    userId: string;
    email: string;
    name: string;
  };
  'order.placed': EventPayload & {
    orderId: string;
    userId: string;
    items: Array<{ productId: string; quantity: number }>;
    total: number;
  };
  'payment.completed': EventPayload & {
    paymentId: string;
    orderId: string;
    amount: number;
  };
}

export type DomainEventName = keyof DomainEvents;
```

```typescript
// src/events/eventBus.ts
import { EventEmitter } from 'events';
import { v4 as uuidv4 } from 'uuid';
import { logger } from '../lib/logger';

class TypedEventBus extends EventEmitter {
  publish<T extends DomainEventName>(
    eventName: T,
    data: Omit<DomainEvents[T], 'eventId' | 'timestamp' | 'version'>
  ): void {
    const payload: DomainEvents[T] = {
      eventId: uuidv4(),
      timestamp: new Date().toISOString(),
      version: 1,
      ...data,
    } as DomainEvents[T];

    logger.info({ eventName, eventId: payload.eventId }, 'Event published');
    this.emit(eventName, payload);
  }

  subscribe<T extends DomainEventName>(
    eventName: T,
    handler: (payload: DomainEvents[T]) => Promise<void>
  ): void {
    this.on(eventName, async (payload: DomainEvents[T]) => {
      try {
        await handler(payload);
      } catch (err) {
        logger.error({ eventName, eventId: payload.eventId, err }, 'Event handler failed');
      }
    });
  }
}

export const eventBus = new TypedEventBus();
```

---

## イベントハンドラーの生成

```
注文完了時に複数のサービスを連鎖させるイベントハンドラーを生成してください。

フロー:
1. OrderService: order.placed イベントを発行
2. InventoryService: order.placed を受信 → 在庫を引く
3. NotificationService: order.placed を受信 → 注文確認メールをキューに追加
4. AnalyticsService: order.placed を受信 → 売上データを記録

要件：
- 各ハンドラーは冪等（orderId で重複チェック）
- エラーはログ記録のみ（ハンドラーの失敗が注文を妨げない）

生成ファイル:
- src/events/handlers/inventoryHandler.ts
- src/events/handlers/notificationHandler.ts
- src/events/handlers/analyticsHandler.ts
```

```typescript
// src/events/handlers/inventoryHandler.ts
export function registerInventoryHandlers(): void {
  eventBus.subscribe('order.placed', async (payload) => {
    // 冪等性チェック
    const processed = await inventoryRepo.isOrderProcessed(payload.orderId);
    if (processed) {
      logger.info({ orderId: payload.orderId }, 'Order already processed, skipping');
      return;
    }

    for (const item of payload.items) {
      await inventoryRepo.decrementStock(item.productId, item.quantity);
    }

    await inventoryRepo.markOrderProcessed(payload.orderId);
    logger.info({ orderId: payload.orderId }, 'Inventory updated');
  });
}
```

---

## Outboxパターンで信頼性を確保する

```
Outboxパターンを使ってイベントの確実な発行を実装してください。

問題: DBの更新とイベント発行の間でクラッシュするとイベントが失われる
解決: DBトランザクション内にoutboxテーブルにイベントを保存 → 別プロセスでポーリングして発行

要件：
- DBトランザクション: Prisma
- outboxテーブル: id, eventName, payload, publishedAt（NULLは未発行）
- ポーリング間隔: 5秒
- 発行成功後: publishedAtを更新
```

---

## まとめ

Claude CodeでイベントDDアーキテクチャを設計する：

1. **CLAUDE.md** にイベント命名規則・ペイロード構造・信頼性要件を明記
2. **型安全なEventBus** でイベント名とペイロードの対応をTypeScriptで保証
3. **冪等なハンドラー** で重複処理を防ぐ
4. **Outboxパターン** でイベント発行の信頼性を確保

---

*イベント設計のコードレビューは **Code Review Pack（¥980）** の `/code-review` で自動化できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
