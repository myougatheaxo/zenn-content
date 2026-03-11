---
title: "Claude CodeでWebhook配信保証を設計する：at-least-once配信・署名検証・リトライキュー"
emoji: "📨"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "security"]
published: true
published_at: "2026-03-16 14:00"
---

## はじめに

「Webhookを送ったのか送っていないのかわからない」「届かなかったときの再送ができない」——at-least-once配信・HMAC署名・指数バックオフリトライでWebhook配信を確実にする設計をClaude Codeに生成させる。

---

## CLAUDE.mdにWebhook配信保証設計ルールを書く

```markdown
## Webhook配信保証設計ルール

### 配信保証
- at-least-once配信（最低1回は必ず届ける）
- 冪等性: 受信側でevent_idによる重複チェックが必要（配信側は保証できない）
- 配信タイムアウト: 10秒（受信側の重い処理を待たない）

### 署名検証
- HMAC-SHA256でペイロードに署名（秘密鍵はユーザーごとに異なる）
- X-Webhook-Signatureヘッダー: sha256=<hex>
- タイムスタンプ検証: X-Webhook-Timestamp（5分以上古いリクエストは拒否）

### リトライ戦略
- 失敗時: 指数バックオフ（1m→5m→15m→1h→4h→12h）
- 最大リトライ: 6回（合計約17時間）
- 6回失敗で配信失敗ログ + オーナーへアラート
```

---

## Webhook配信保証の生成

```
Webhook配信保証システムを設計してください。

要件：
- at-least-once配信（BullMQキュー）
- HMAC署名付与
- タイムスタンプ検証
- 指数バックオフリトライ
- 配信ログ記録

生成ファイル: src/webhooks/
```

---

## 生成されるWebhook配信保証実装

```typescript
// src/webhooks/signer.ts — HMAC署名

import crypto from 'crypto';

export function signWebhookPayload(
  payload: string,         // JSON.stringify済み文字列
  secret: string,          // ユーザーごとの秘密鍵
  timestamp: number        // Unix秒
): { signature: string; signedPayload: string } {
  // 署名対象: タイムスタンプ + ペイロード（タイムスタンプを含めてリプレイ攻撃を防止）
  const signedPayload = `${timestamp}.${payload}`;
  const signature = crypto
    .createHmac('sha256', secret)
    .update(signedPayload)
    .digest('hex');

  return { signature: `sha256=${signature}`, signedPayload };
}

// 受信側: 署名検証（受信者が実装）
export function verifyWebhookSignature(
  payload: string,
  timestamp: string,
  signature: string,
  secret: string,
  toleranceSeconds = 300 // 5分以内のリクエストのみ受け付ける
): boolean {
  // タイムスタンプ検証（リプレイ攻撃防止）
  const ts = parseInt(timestamp, 10);
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - ts) > toleranceSeconds) return false;

  const signedPayload = `${timestamp}.${payload}`;
  const expected = crypto
    .createHmac('sha256', secret)
    .update(signedPayload)
    .digest('hex');

  const expected256 = `sha256=${expected}`;
  // タイミングセーフ比較（timing attack防止）
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected256));
}
```

```typescript
// src/webhooks/deliveryService.ts — Webhook配信サービス

const RETRY_DELAYS_MS = [
  60_000,         // 1分後
  300_000,        // 5分後
  900_000,        // 15分後
  3_600_000,      // 1時間後
  14_400_000,     // 4時間後
  43_200_000,     // 12時間後
];

export class WebhookDeliveryService {
  async enqueue(event: WebhookEvent): Promise<void> {
    // 配信対象のエンドポイントを取得
    const endpoints = await prisma.webhookEndpoint.findMany({
      where: {
        tenantId: event.tenantId,
        events: { has: event.type }, // 購読しているイベントタイプのみ
        active: true,
      },
    });

    // 各エンドポイントへのジョブをキューに追加
    for (const endpoint of endpoints) {
      const job = await prisma.webhookDelivery.create({
        data: {
          eventId: event.id,
          endpointId: endpoint.id,
          eventType: event.type,
          payload: JSON.stringify(event.data),
          status: 'pending',
          attempt: 0,
          maxAttempts: RETRY_DELAYS_MS.length + 1,
        },
      });

      await deliveryQueue.add('deliver', { deliveryId: job.id }, {
        attempts: 1, // BullMQのリトライは使わず独自管理
      });
    }
  }

  async deliver(deliveryId: string): Promise<void> {
    const delivery = await prisma.webhookDelivery.findUniqueOrThrow({
      where: { id: deliveryId },
      include: { endpoint: true },
    });

    const timestamp = Math.floor(Date.now() / 1000);
    const { signature } = signWebhookPayload(delivery.payload, delivery.endpoint.secret, timestamp);

    const startMs = Date.now();
    let response: Response;

    try {
      // タイムアウト10秒
      response = await fetch(delivery.endpoint.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Event': delivery.eventType,
          'X-Webhook-Delivery': delivery.id,
          'X-Webhook-Timestamp': timestamp.toString(),
          'X-Webhook-Signature': signature,
        },
        body: delivery.payload,
        signal: AbortSignal.timeout(10_000),
      });
    } catch (err) {
      // ネットワークエラー / タイムアウト
      await this.handleDeliveryFailure(delivery, err as Error);
      return;
    }

    const durationMs = Date.now() - startMs;

    await prisma.webhookDelivery.update({
      where: { id: deliveryId },
      data: {
        attempt: { increment: 1 },
        lastAttemptAt: new Date(),
        lastResponseCode: response.status,
        lastResponseBody: (await response.text()).slice(0, 1000),
        durationMs,
        status: response.ok ? 'delivered' : 'failed',
      },
    });

    if (!response.ok) {
      await this.handleDeliveryFailure(delivery, new Error(`HTTP ${response.status}`));
    } else {
      logger.info({ deliveryId, durationMs, status: response.status }, 'Webhook delivered');
    }
  }

  private async handleDeliveryFailure(
    delivery: WebhookDelivery & { endpoint: WebhookEndpoint },
    error: Error
  ): Promise<void> {
    const nextAttempt = delivery.attempt;

    if (nextAttempt >= RETRY_DELAYS_MS.length) {
      // 最大リトライ超過 → 配信失敗確定
      await prisma.webhookDelivery.update({
        where: { id: delivery.id },
        data: { status: 'exhausted' },
      });

      // オーナーへアラート
      await notificationService.notifyWebhookFailure(delivery.endpoint.tenantId, {
        endpointUrl: delivery.endpoint.url,
        eventType: delivery.eventType,
        attempts: nextAttempt + 1,
      });

      logger.error({ deliveryId: delivery.id }, 'Webhook delivery exhausted');
      return;
    }

    // リトライスケジュール
    const delayMs = RETRY_DELAYS_MS[nextAttempt];
    await deliveryQueue.add(
      'deliver',
      { deliveryId: delivery.id },
      { delay: delayMs }
    );

    logger.warn({ deliveryId: delivery.id, attempt: nextAttempt, delayMs, error: error.message }, 'Webhook delivery failed, retrying');
  }
}
```

```typescript
// src/webhooks/receiver.ts — Webhook受信側の実装例（自社サービス内での参考）

router.post('/webhooks/stripe', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-webhook-signature'] as string;
  const timestamp = req.headers['x-webhook-timestamp'] as string;
  const payload = req.body.toString();

  // 署名検証
  const isValid = verifyWebhookSignature(payload, timestamp, signature, process.env.WEBHOOK_SECRET!);
  if (!isValid) {
    logger.warn('Invalid webhook signature');
    return res.sendStatus(401);
  }

  const event = JSON.parse(payload);

  // 冪等性チェック（重複配信対応）
  const deliveryId = req.headers['x-webhook-delivery'] as string;

  processedDeliveries.setIfAbsent(deliveryId, async () => {
    // 実際のイベント処理（冪等な実装）
    await handleWebhookEvent(event);
  }).then(() => res.sendStatus(200));
});
```

---

## まとめ

Claude CodeでWebhook配信保証を設計する：

1. **CLAUDE.md** にat-least-once配信・HMAC-SHA256署名・5分タイムスタンプ許容・6回リトライ（最大17時間）を明記
2. **署名ペイロードにタイムスタンプを含める** ——同じpayloadで古い署名を再利用するリプレイ攻撃を防止
3. **タイムアウト10秒** で受信側の重い処理をブロックしない——配信側は応答だけ確認し実際の処理は受信側が非同期で行う
4. **6回失敗で配信失敗確定+オーナーアラート** ——サイレント消失させず、受信エンドポイントの障害を検出してユーザーに通知

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
