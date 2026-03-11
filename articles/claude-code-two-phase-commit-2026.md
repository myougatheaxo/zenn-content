---
title: "Claude Codeで2フェーズコミット代替パターンを設計する：分散トランザクションの調整・補償・整合性保証"
emoji: "🔗"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-20 10:00"
---

## はじめに

「マイクロサービス間でDBトランザクションが使えない」「注文確定とポイント付与が別サービスで片方だけ成功する」——2フェーズコミット（2PC）の問題点を理解し、Sagaパターン・TCCパターン・Outboxパターンで分散整合性を実現する設計をClaude Codeに生成させる。

---

## CLAUDE.mdに分散トランザクション設計ルールを書く

```markdown
## 分散トランザクション設計ルール

### なぜ2PCを避けるか
- コーディネーター障害でデッドロック（blocking protocol）
- マイクロサービス境界を越えるXAトランザクションは可用性を下げる
- 推奨: Saga / TCC / Outboxパターンで結果整合

### TCCパターン（Try-Confirm-Cancel）
- Try: リソースを仮予約（在庫仮押さえ・残高仮確保）
- Confirm: 全Try成功後に確定
- Cancel: いずれかのTry失敗後に仮予約を解放
- 各サービスが独立してTry/Confirm/Cancelを実装

### 補償トランザクション
- すでにコミットされた操作を論理的に取り消す
- 時間窓あり（例: 支払い済みはN時間以内にキャンセル可）
- 補償できない操作はフォワードリカバリー（再試行）
```

---

## 分散トランザクション実装の生成

```
2フェーズコミット代替パターンを実装してください。

要件：
- TCCパターンの実装
- 補償トランザクション
- タイムアウトと自動補償
- 整合性チェック

生成ファイル: src/distributed/
```

---

## 生成される分散トランザクション実装

```typescript
// src/distributed/tcc/tccCoordinator.ts — TCCコーディネーター

export type TccPhase = 'try' | 'confirm' | 'cancel' | 'completed' | 'cancelled' | 'failed';

export interface TccTransaction {
  txId: string;
  phase: TccPhase;
  participants: TccParticipant[];
  createdAt: Date;
  confirmedAt?: Date;
  cancelledAt?: Date;
  timeout: number;  // ミリ秒
}

export interface TccParticipant {
  serviceId: string;
  reservationId: string;  // Tryで確保したリソースのID
  status: 'tried' | 'confirmed' | 'cancelled' | 'failed';
  tryAt?: Date;
  confirmAt?: Date;
  cancelAt?: Date;
}

export interface ITccService {
  serviceId: string;
  try(input: unknown): Promise<{ reservationId: string }>;
  confirm(reservationId: string): Promise<void>;
  cancel(reservationId: string): Promise<void>;
}

export class TccCoordinator {
  constructor(
    private readonly txStore: ITccTransactionStore,
    private readonly services: Map<string, ITccService>
  ) {}

  async execute<T>(
    steps: Array<{ serviceId: string; input: unknown }>,
    options: { timeout?: number } = {}
  ): Promise<void> {
    const txId = ulid();
    const timeout = options.timeout ?? 30_000;

    const tx = await this.txStore.create({
      txId,
      phase: 'try',
      participants: [],
      createdAt: new Date(),
      timeout,
    });

    // Phase 1: Try（全サービスへ仮予約）
    const tryResults: TccParticipant[] = [];
    let tryError: Error | null = null;

    for (const step of steps) {
      const service = this.services.get(step.serviceId);
      if (!service) throw new Error(`Service ${step.serviceId} not registered`);

      try {
        const { reservationId } = await Promise.race([
          service.try(step.input),
          this.timeoutPromise(timeout, `Try timeout: ${step.serviceId}`),
        ]);

        const participant: TccParticipant = {
          serviceId: step.serviceId,
          reservationId,
          status: 'tried',
          tryAt: new Date(),
        };
        tryResults.push(participant);
      } catch (error) {
        tryError = error as Error;
        logger.error({ txId, serviceId: step.serviceId, error }, 'TCC Try failed');
        break;
      }
    }

    await this.txStore.updateParticipants(txId, tryResults);

    if (tryError || tryResults.length < steps.length) {
      // Phase 1失敗 → Cancel
      await this.txStore.updatePhase(txId, 'cancel');
      await this.cancelAll(txId, tryResults);
      await this.txStore.updatePhase(txId, 'cancelled');
      throw new TccTransactionError(
        txId,
        `TCC transaction failed in Try phase: ${tryError?.message}`
      );
    }

    // Phase 2: Confirm（全サービスへ確定）
    await this.txStore.updatePhase(txId, 'confirm');
    const confirmErrors: Error[] = [];

    for (const participant of tryResults) {
      const service = this.services.get(participant.serviceId)!;
      try {
        await Promise.race([
          service.confirm(participant.reservationId),
          this.timeoutPromise(timeout, `Confirm timeout: ${participant.serviceId}`),
        ]);
        participant.status = 'confirmed';
        participant.confirmAt = new Date();
      } catch (error) {
        // Confirm失敗は自動リトライ（最終的には必ず成功させる必要がある）
        confirmErrors.push(error as Error);
        logger.error({ txId, participant, error }, 'TCC Confirm failed - scheduling retry');
        await this.scheduleConfirmRetry(txId, participant.serviceId, participant.reservationId);
      }
    }

    await this.txStore.updateParticipants(txId, tryResults);

    if (confirmErrors.length > 0) {
      // Confirmは失敗してもforward recovery（リトライ）
      await this.txStore.updatePhase(txId, 'failed');
      throw new TccTransactionError(txId, 'Confirm phase partially failed, retrying asynchronously');
    }

    await this.txStore.updatePhase(txId, 'completed');
    logger.info({ txId }, 'TCC transaction completed');
  }

  private async cancelAll(txId: string, participants: TccParticipant[]): Promise<void> {
    // 逆順にCancel（依存関係を考慮）
    const reversed = [...participants].reverse();

    await Promise.allSettled(
      reversed.map(async (participant) => {
        const service = this.services.get(participant.serviceId);
        if (!service) return;
        try {
          await service.cancel(participant.reservationId);
          participant.status = 'cancelled';
          participant.cancelAt = new Date();
        } catch (error) {
          // Cancel失敗はDLQへ（人間の介入が必要な場合がある）
          logger.error({ txId, participant, error }, 'TCC Cancel failed - manual intervention required');
          await this.deadLetterQueue.push({ txId, participant, error: String(error) });
        }
      })
    );
  }

  private async scheduleConfirmRetry(
    txId: string,
    serviceId: string,
    reservationId: string
  ): Promise<void> {
    // 指数バックオフでリトライをスケジュール
    const retryDelays = [1000, 5000, 30000, 120000, 600000];
    for (const delay of retryDelays) {
      await redis.zAdd('tcc:confirm:retry', {
        score: Date.now() + delay,
        value: JSON.stringify({ txId, serviceId, reservationId, delay }),
      });
    }
  }

  private timeoutPromise<T>(ms: number, message: string): Promise<T> {
    return new Promise((_, reject) =>
      setTimeout(() => reject(new TccTimeoutError(message)), ms)
    );
  }
}
```

```typescript
// 在庫サービスのTCC実装例
export class InventoryTccService implements ITccService {
  readonly serviceId = 'inventory';

  // Try: 在庫を仮押さえ
  async try(input: { productId: string; quantity: number }): Promise<{ reservationId: string }> {
    return prisma.$transaction(async (tx) => {
      const inventory = await tx.inventory.findUnique({
        where: { productId: input.productId },
      });

      if (!inventory || inventory.available < input.quantity) {
        throw new InsufficientStockError(input.productId, input.quantity, inventory?.available ?? 0);
      }

      // 仮押さえ（実際の在庫は減らさず、reservedを増やす）
      await tx.inventory.update({
        where: { productId: input.productId },
        data: {
          available: { decrement: input.quantity },
          reserved: { increment: input.quantity },
        },
      });

      const reservation = await tx.stockReservation.create({
        data: {
          id: ulid(),
          productId: input.productId,
          quantity: input.quantity,
          status: 'reserved',
          expiresAt: new Date(Date.now() + 10 * 60 * 1000),  // 10分で自動解放
        },
      });

      return { reservationId: reservation.id };
    });
  }

  // Confirm: 仮押さえを確定（reservedを減らして在庫を確定的に減らす）
  async confirm(reservationId: string): Promise<void> {
    await prisma.$transaction(async (tx) => {
      const reservation = await tx.stockReservation.findUnique({
        where: { id: reservationId },
      });
      if (!reservation || reservation.status !== 'reserved') {
        throw new ReservationNotFoundError(reservationId);
      }

      await tx.inventory.update({
        where: { productId: reservation.productId },
        data: { reserved: { decrement: reservation.quantity } },
      });

      await tx.stockReservation.update({
        where: { id: reservationId },
        data: { status: 'confirmed', confirmedAt: new Date() },
      });
    });
  }

  // Cancel: 仮押さえを解放
  async cancel(reservationId: string): Promise<void> {
    await prisma.$transaction(async (tx) => {
      const reservation = await tx.stockReservation.findUnique({
        where: { id: reservationId },
      });
      if (!reservation) return;  // 冪等性: すでにキャンセル済みは成功扱い
      if (reservation.status === 'confirmed') {
        throw new AlreadyConfirmedError(reservationId);
      }

      await tx.inventory.update({
        where: { productId: reservation.productId },
        data: {
          available: { increment: reservation.quantity },
          reserved: { decrement: reservation.quantity },
        },
      });

      await tx.stockReservation.update({
        where: { id: reservationId },
        data: { status: 'cancelled', cancelledAt: new Date() },
      });
    });
  }
}

// TCCを使った注文フロー
const coordinator = new TccCoordinator(txStore, new Map([
  ['inventory', new InventoryTccService()],
  ['payment', new PaymentTccService()],
  ['loyalty', new LoyaltyTccService()],
]));

export class PlaceOrderWithTccUseCase {
  async execute(input: PlaceOrderInput): Promise<string> {
    const orderId = ulid();

    await coordinator.execute([
      { serviceId: 'inventory', input: { productId: input.productId, quantity: input.quantity } },
      { serviceId: 'payment', input: { userId: input.userId, amount: input.totalAmount } },
      { serviceId: 'loyalty', input: { userId: input.userId, points: input.earnedPoints } },
    ], { timeout: 15_000 });

    await orderRepository.save(Order.create(orderId, input));
    return orderId;
  }
}
```

---

## まとめ

Claude Codeで2フェーズコミット代替パターンを設計する：

1. **CLAUDE.md** に2PCはコーディネーター障害でデッドロック・TCCパターン（Try仮予約→Confirm確定→Cancel解放）で結果整合・ConfirmはForward Recovery（必ずリトライ）を明記
2. **Try=仮押さえ** でビジネス的に安全な予約——在庫の`available`を減らし`reserved`を増やすだけ。全サービスのTryが成功するまで実際の確定はしない。失敗したらCancelで元に戻す
3. **Confirm失敗はForward Recovery** ——ConfirmはDBにすでに反映されたものを確定する操作なのでCancelできない。失敗したら指数バックオフでリトライ。それでも失敗したら人間の介入（DLQ）
4. **仮予約の有効期限（10分）** で自動クリーンアップ——TCCコーディネーターが死んでも、expiresAtを過ぎた仮予約をバッチで自動解放。ゾンビ予約が在庫を永久に占有しない

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
