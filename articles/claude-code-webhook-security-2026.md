---
title: "Claude CodeでWebhookセキュリティを設計する：HMAC署名検証・リプレイ攻撃防止・配信保証"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-21 20:00"
---

## はじめに

「誰でもWebhookエンドポイントに偽のリクエストを送れる」「同じWebhookイベントが複数回届いて二重処理が発生した」——HMAC署名検証・タイムスタンプ検証・冪等処理でWebhookを安全に受信する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにWebhookセキュリティ設計ルールを書く

```markdown
## Webhookセキュリティ設計ルール

### 署名検証
- 全Webhookリクエストに署名を検証（未署名は拒否）
- HMAC-SHA256でシークレットキーと本文をハッシュ
- タイムスタンプ検証（5分以上古いリクエストを拒否）

### リプレイ攻撃防止
- タイムスタンプ+IDでリプレイを検知
- 受信済みイベントIDをRedisに記録（TTL: 10分）
- 同一イベントIDは2回目以降は処理をスキップ

### 配信保証
- Webhook受信後は即座に202 Acceptedを返す
- 処理は非同期で実行（タイムアウトを防ぐ）
- 処理失敗時は自動リトライ（DLQ対応）
```

---

## Webhookセキュリティ実装の生成

```
Webhookセキュリティを設計してください。

要件：
- HMAC-SHA256署名検証
- リプレイ攻撃防止
- 冪等処理
- 配信保証

生成ファイル: src/webhooks/
```

---

## 生成されるWebhookセキュリティ実装

```typescript
// src/webhooks/webhookVerifier.ts — Webhook署名検証

import { createHmac, timingSafeEqual } from 'crypto';

export interface WebhookVerificationOptions {
  secret: string;           // HMAC署名のシークレットキー
  timestampHeader: string;  // タイムスタンプヘッダー名
  signatureHeader: string;  // 署名ヘッダー名
  toleranceSeconds: number; // タイムスタンプ許容範囲（秒）
}

export class WebhookVerifier {
  constructor(private readonly options: WebhookVerificationOptions) {}

  verify(rawBody: Buffer, headers: Record<string, string>): void {
    // 1. タイムスタンプ検証
    const timestamp = headers[this.options.timestampHeader];
    if (!timestamp) {
      throw new WebhookVerificationError('Missing timestamp header');
    }

    const requestTime = parseInt(timestamp);
    if (isNaN(requestTime)) {
      throw new WebhookVerificationError('Invalid timestamp format');
    }

    const now = Math.floor(Date.now() / 1000);
    const age = Math.abs(now - requestTime);

    if (age > this.options.toleranceSeconds) {
      throw new WebhookVerificationError(
        `Timestamp too old or in the future: ${age}s (tolerance: ${this.options.toleranceSeconds}s)`
      );
    }

    // 2. 署名検証
    const signature = headers[this.options.signatureHeader];
    if (!signature) {
      throw new WebhookVerificationError('Missing signature header');
    }

    // 署名ペイロード: timestamp + "." + body
    const payload = `${timestamp}.${rawBody.toString('utf8')}`;
    const expectedSignature = 'sha256=' + createHmac('sha256', this.options.secret)
      .update(payload)
      .digest('hex');

    // タイミング攻撃を防ぐためにtimingSafeEqualを使用
    const sigBuffer = Buffer.from(signature, 'utf8');
    const expectedBuffer = Buffer.from(expectedSignature, 'utf8');

    if (sigBuffer.length !== expectedBuffer.length || !timingSafeEqual(sigBuffer, expectedBuffer)) {
      throw new WebhookVerificationError('Invalid signature');
    }
  }
}

// Stripeスタイルの署名ヘッダー解析
export function parseStripeSignatureHeader(header: string): {
  timestamp: string;
  signatures: string[];
} {
  const parts = header.split(',');
  const timestamp = parts.find(p => p.startsWith('t='))?.slice(2) ?? '';
  const signatures = parts
    .filter(p => p.startsWith('v1='))
    .map(p => p.slice(3));

  return { timestamp, signatures };
}
```

```typescript
// src/webhooks/webhookProcessor.ts — 冪等Webhook処理

export class IdempotentWebhookProcessor {
  private readonly PROCESSED_KEY = (eventId: string) => `webhook:processed:${eventId}`;
  private readonly PROCESSED_TTL = 10 * 60;  // 10分（タイムスタンプ許容範囲の2倍）

  constructor(
    private readonly redis: Redis,
    private readonly verifier: WebhookVerifier
  ) {}

  // Webhook受信エンドポイントで使用
  async process<T>(
    rawBody: Buffer,
    headers: Record<string, string>,
    eventId: string,
    handler: (payload: T) => Promise<void>
  ): Promise<{ processed: boolean; reason?: string }> {
    // 1. 署名検証（改ざん防止）
    this.verifier.verify(rawBody, headers);

    // 2. 冪等チェック（重複処理防止）
    const key = this.PROCESSED_KEY(eventId);
    const alreadyProcessed = await this.redis.setNX(key, new Date().toISOString());

    if (!alreadyProcessed) {
      logger.info({ eventId }, 'Webhook event already processed, skipping');
      return { processed: false, reason: 'duplicate' };
    }

    // TTLを設定（自動クリーンアップ）
    await this.redis.expire(key, this.PROCESSED_TTL);

    // 3. ペイロードをパース
    const payload: T = JSON.parse(rawBody.toString('utf8'));

    // 4. ハンドラーを実行
    try {
      await handler(payload);
      logger.info({ eventId }, 'Webhook event processed successfully');
      return { processed: true };
    } catch (error) {
      // 処理失敗時はRedisから削除（リトライできるように）
      await this.redis.del(key);
      throw error;
    }
  }
}

// Express Webhook受信エンドポイント
const stripeVerifier = new WebhookVerifier({
  secret: process.env.STRIPE_WEBHOOK_SECRET!,
  timestampHeader: 'stripe-signature',  // Stripeは一つのヘッダーにtとv1を含む
  signatureHeader: 'stripe-signature',
  toleranceSeconds: 300,  // 5分
});

const webhookProcessor = new IdempotentWebhookProcessor(redis, stripeVerifier);

// rawBodyが必要（署名計算のため）
app.post('/webhooks/stripe',
  express.raw({ type: 'application/json' }),  // rawBodyを保持
  async (req, res, next) => {
    try {
      const signatureHeader = req.headers['stripe-signature'] as string;
      const { timestamp, signatures } = parseStripeSignatureHeader(signatureHeader);

      // イベントIDをStripeのペイロードから取得
      const rawBody = req.body as Buffer;
      const tempPayload = JSON.parse(rawBody.toString('utf8'));
      const eventId = tempPayload.id;

      const result = await webhookProcessor.process(
        rawBody,
        { 'stripe-signature': signatureHeader },
        eventId,
        async (payload: StripeEvent) => {
          switch (payload.type) {
            case 'payment_intent.succeeded':
              await orderService.confirmPayment(payload.data.object.metadata.orderId);
              break;
            case 'payment_intent.payment_failed':
              await orderService.failPayment(payload.data.object.metadata.orderId);
              break;
          }
        }
      );

      // 即座に202を返す（処理は非同期）
      res.status(202).json({ received: true, processed: result.processed });
    } catch (error) {
      if (error instanceof WebhookVerificationError) {
        logger.warn({ error: error.message }, 'Webhook verification failed');
        return res.status(400).json({ error: 'Invalid webhook signature' });
      }
      next(error);
    }
  }
);

// Webhook送信側（HMAC署名を付与）
export function signWebhookPayload(payload: unknown, secret: string): {
  signature: string;
  timestamp: number;
} {
  const timestamp = Math.floor(Date.now() / 1000);
  const body = JSON.stringify(payload);
  const signingPayload = `${timestamp}.${body}`;

  const signature = 'sha256=' + createHmac('sha256', secret)
    .update(signingPayload)
    .digest('hex');

  return { signature, timestamp };
}
```

---

## まとめ

Claude CodeでWebhookセキュリティを設計する：

1. **CLAUDE.md** にHMAC-SHA256署名検証を必須・タイムスタンプ許容5分（リプレイ攻撃防止）・RedisでイベントID冪等チェック・受信後即202（処理は非同期）を明記
2. **`timingSafeEqual()`で署名比較** ——通常の文字列比較は最初の不一致で即座にfalseを返すためタイミング攻撃が可能。`timingSafeEqual`は常に一定時間かけて比較するためシークレットの内容が漏れない
3. **タイムスタンプ検証（5分以内）でリプレイ攻撃を防止** ——署名が正しくても5分以上前のリクエストを拒否。攻撃者が正規のWebhookを録画して後で再送する攻撃（リプレイ攻撃）を防ぐ
4. **RedisでイベントID冪等チェック** ——`SET NX`でイベントIDを記録し、2回目は処理をスキップ。Webhookプロバイダーは同一イベントを複数回送ることがある（ネットワーク問題等）ため二重処理を防ぐ必須の対策

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
