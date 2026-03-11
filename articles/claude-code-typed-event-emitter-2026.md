---
title: "Claude Codeで型安全なイベントエミッターを設計する：TypeScript EventEmitter・型付きイベントマップ・メモリリーク防止"
emoji: "📡"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture"]
published: true
published_at: "2026-03-21 10:00"
---

## はじめに

「EventEmitterのイベント名をタイポしてリスナーが呼ばれない」「emitする引数の型とonで受け取る型が合っていない」——TypeScriptで型安全なイベントエミッターを設計し、コンパイル時にイベント名と引数型を保証する設計をClaude Codeに生成させる。

---

## CLAUDE.mdに型安全イベントエミッター設計ルールを書く

```markdown
## 型安全イベントエミッター設計ルール

### イベントマップの定義
- EventMap型でイベント名とペイロード型を一元定義
- TypedEventEmitter<EventMap>で型推論を強制
- イベント名のタイポはコンパイルエラー

### メモリリーク防止
- リスナーの最大数を設定（defaultMaxListeners）
- コンポーネント/リクエストのライフサイクルでクリーンアップ
- AbortSignalと組み合わせてリスナーを自動解除

### 非同期イベント
- 非同期リスナーのエラーはcatchしてログに記録
- 非同期イベントはasync/awaitでシリアル実行 or Promiseで並列実行
```

---

## 型安全イベントエミッター実装の生成

```
TypeScript型安全イベントエミッターを設計してください。

要件：
- イベントマップによる型安全
- ワイルドカードリスナー
- once/onの型推論
- メモリリーク検出

生成ファイル: src/shared/events/
```

---

## 生成される型安全イベントエミッター実装

```typescript
// src/shared/events/typedEventEmitter.ts — 型安全イベントエミッター

// イベントマップの型定義パターン
export type EventMap = Record<string, unknown>;

// 型安全なイベントエミッター
export class TypedEventEmitter<TEventMap extends EventMap> {
  private readonly listeners = new Map<
    keyof TEventMap,
    Set<(payload: unknown) => void | Promise<void>>
  >();
  private readonly onceListeners = new Map<
    keyof TEventMap,
    Set<(payload: unknown) => void | Promise<void>>
  >();
  private maxListeners: number = 10;

  // イベントリスナーを登録
  on<K extends keyof TEventMap>(
    event: K,
    listener: (payload: TEventMap[K]) => void | Promise<void>
  ): () => void {  // クリーンアップ関数を返す
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }

    const set = this.listeners.get(event)!;

    if (set.size >= this.maxListeners) {
      console.warn(
        `MaxListenersExceededWarning: ${set.size + 1} listeners added for event "${String(event)}". ` +
        `This may be a memory leak. Use setMaxListeners() to increase the limit.`
      );
    }

    set.add(listener as (payload: unknown) => void);

    // クリーンアップ関数（off()の代わりに使える）
    return () => this.off(event, listener);
  }

  // 一度だけ実行するリスナー
  once<K extends keyof TEventMap>(
    event: K,
    listener: (payload: TEventMap[K]) => void | Promise<void>
  ): () => void {
    if (!this.onceListeners.has(event)) {
      this.onceListeners.set(event, new Set());
    }
    const set = this.onceListeners.get(event)!;
    set.add(listener as (payload: unknown) => void);

    return () => set.delete(listener as (payload: unknown) => void);
  }

  // リスナーを削除
  off<K extends keyof TEventMap>(
    event: K,
    listener: (payload: TEventMap[K]) => void | Promise<void>
  ): void {
    this.listeners.get(event)?.delete(listener as (payload: unknown) => void);
    this.onceListeners.get(event)?.delete(listener as (payload: unknown) => void);
  }

  // イベントを発火
  async emit<K extends keyof TEventMap>(event: K, payload: TEventMap[K]): Promise<void> {
    const listeners = this.listeners.get(event) ?? new Set();
    const onceListeners = this.onceListeners.get(event) ?? new Set();

    // onceリスナーは呼ぶ前に削除（重複実行防止）
    this.onceListeners.set(event, new Set());

    const allListeners = [...listeners, ...onceListeners];

    // 非同期リスナーのエラーを個別にキャッチ（1つの失敗が他に影響しない）
    await Promise.allSettled(
      allListeners.map(async (listener) => {
        try {
          await listener(payload);
        } catch (error) {
          logger.error({ event: String(event), error }, 'Event listener threw an error');
        }
      })
    );
  }

  // AbortSignalと連携した自動クリーンアップ
  onAbortable<K extends keyof TEventMap>(
    event: K,
    listener: (payload: TEventMap[K]) => void,
    signal: AbortSignal
  ): void {
    const cleanup = this.on(event, listener);
    signal.addEventListener('abort', cleanup, { once: true });
  }

  // Promiseとして待機（once + timeout）
  waitFor<K extends keyof TEventMap>(
    event: K,
    timeoutMs?: number
  ): Promise<TEventMap[K]> {
    return new Promise((resolve, reject) => {
      const cleanup = this.once(event, resolve as (payload: TEventMap[K]) => void);

      if (timeoutMs) {
        const timer = setTimeout(() => {
          cleanup();
          reject(new Error(`Timeout waiting for event: ${String(event)}`));
        }, timeoutMs);

        this.once(event, () => clearTimeout(timer));
      }
    });
  }

  setMaxListeners(n: number): void { this.maxListeners = n; }
  listenerCount<K extends keyof TEventMap>(event: K): number {
    return (this.listeners.get(event)?.size ?? 0) + (this.onceListeners.get(event)?.size ?? 0);
  }
}
```

```typescript
// src/domain/order/orderEvents.ts — ドメインイベントマップ定義

// イベントマップで型を一元管理
export interface OrderEventMap {
  'order:created': {
    orderId: string;
    userId: string;
    total: number;
    items: Array<{ productId: string; quantity: number }>;
  };
  'order:submitted': {
    orderId: string;
    userId: string;
    paymentIntentId: string;
  };
  'order:completed': {
    orderId: string;
    userId: string;
    completedAt: Date;
  };
  'order:cancelled': {
    orderId: string;
    userId: string;
    reason: string;
  };
  'order:item:added': {
    orderId: string;
    productId: string;
    quantity: number;
  };
}

// シングルトンイベントエミッター
export const orderEvents = new TypedEventEmitter<OrderEventMap>();

// 型安全な使用例
orderEvents.on('order:created', async (payload) => {
  // payloadは { orderId: string; userId: string; total: number; items: ... } と型推論される
  logger.info({ orderId: payload.orderId }, 'Order created');
  await notificationService.sendOrderConfirmation(payload.userId, payload.orderId);
});

orderEvents.on('order:completed', async (payload) => {
  await loyaltyService.addPoints(payload.userId, payload.orderId);
  await inventoryService.confirmDeduction(payload.orderId);
});

// ❌ コンパイルエラー: タイポしたイベント名
// orderEvents.on('order:craeted', () => {});  // Property 'order:craeted' does not exist

// ❌ コンパイルエラー: 間違った引数型
// orderEvents.emit('order:created', { orderId: 'x', userId: 'y' });  // total と items が必要

// ✅ 正しい使用
orderEvents.emit('order:created', {
  orderId: order.id,
  userId: order.userId,
  total: order.total.amount,
  items: order.items.map(i => ({ productId: i.productId, quantity: i.quantity })),
});

// waitFor: イベントが発生するまで待機（E2Eテストで有用）
const completedEvent = await orderEvents.waitFor('order:completed', 30_000);
console.log(`Order ${completedEvent.orderId} completed`);

// AbortSignalでリスナーのライフサイクル管理（リクエストスコープ）
app.post('/api/orders/:id/watch', (req, res) => {
  const abortController = new AbortController();
  req.on('close', () => abortController.abort());

  // リクエスト終了時に自動でリスナーが削除される
  orderEvents.onAbortable('order:completed', (payload) => {
    if (payload.orderId === req.params.id) {
      res.json(payload);
      abortController.abort();
    }
  }, abortController.signal);
});
```

---

## まとめ

Claude Codeで型安全なイベントエミッターを設計する：

1. **CLAUDE.md** にEventMapインターフェースでイベント名とペイロード型を一元定義・TypedEventEmitter<T>で型推論を強制・イベント名タイポはコンパイルエラーを明記
2. **EventMapインターフェース** でイベント名とペイロードを一箇所に定義——`OrderEventMap`に全イベントを列挙することで、`on()`・`emit()`が完全な型安全になる。VSCodeで補完も効く
3. **`on()`がクリーンアップ関数を返す** ——`const cleanup = orderEvents.on('order:created', listener)`→`cleanup()`でリスナー削除。`removeEventListener(event, listener)`より忘れにくく、クロージャの参照も不要
4. **`onAbortable(signal)`** でリクエストスコープのリスナーを自動クリーンアップ——ロングポーリングやSSEで「リクエストが切れたらリスナーも消す」を1行で実現。`close`イベントにAbortControllerを紐付けるだけ

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
