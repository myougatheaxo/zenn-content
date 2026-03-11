---
title: "Claude CodeでWebhook受信を設計する：署名検証・冪等性・リトライ処理"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "webhook", "stripe"]
published: true
---

## はじめに

StripeやGitHubからのWebhookを受け取るとき、署名検証を実装しないと偽リクエストを受け入れてしまう。冪等性を実装しないと同じ支払いを二重処理してしまう。Claude Codeに安全なWebhook受信を設計させる。

---

## CLAUDE.mdにWebhookルールを書く

```markdown
## Webhook受信ルール

### セキュリティ（必須）
- 全てのWebhookで署名を検証する（署名なしのリクエストは拒否）
- Stripe: stripe-signatureヘッダーをWebhook Secretで検証
- GitHub: X-Hub-Signature-256でHMAC-SHA256を検証
- 検証は rawBody（文字列）で行う（JSONパース後では不正確）

### 冪等性（必須）
- 全てのWebhookハンドラーは冪等（同じイベントを2回受けても安全）
- idempotencyキー: {provider}-{eventId} でDBに処理済みか確認
- 処理済みのイベントは200を返して無視する

### レスポンス
- Webhook受信後は200を素早く返す（5秒以内）
- 重い処理はBullMQのキューに投げてから返す

### リトライ対策
- 冪等性テーブルで二重処理を防ぐ
- 失敗時のリトライは送信側に任せる
```

---

## Stripe Webhookの生成

```
Stripe Webhookを安全に受信するエンドポイントを生成してください。

対象イベント:
- payment_intent.succeeded → 注文を完了状態にする
- payment_intent.payment_failed → 注文を失敗状態にする
- customer.subscription.deleted → サブスクリプションを停止する

要件：
- stripe-signatureで署名を検証
- rawBodyでの検証（Expressの場合はJSON parserをスキップ）
- 処理済みイベントはDBに記録して二重処理を防ぐ
- 重い処理はBullMQに投げる

生成するファイル:
- src/webhooks/stripeWebhook.ts
- src/webhooks/handlers/paymentHandler.ts
```

---

## 生成されるStripe Webhook

```typescript
// src/webhooks/stripeWebhook.ts
import Stripe from 'stripe';
import { Router } from 'express';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

// 重要: このルートより前にexpress.json()を設定しない
// rawBodyが必要なため
export const stripeWebhookRouter = Router();

stripeWebhookRouter.post(
  '/webhooks/stripe',
  express.raw({ type: 'application/json' }), // JSONパース前のrawBodyを取得
  async (req, res) => {
    const sig = req.headers['stripe-signature'] as string;

    let event: Stripe.Event;
    try {
      // 署名検証（rawBodyが必要）
      event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
    } catch (err) {
      logger.warn({ err }, 'Stripe webhook signature verification failed');
      return res.status(400).send('Signature verification failed');
    }

    // 冪等性チェック
    const processed = await prisma.webhookEvent.findUnique({
      where: { id: `stripe-${event.id}` },
    });
    if (processed) {
      logger.info({ eventId: event.id }, 'Webhook already processed, skipping');
      return res.status(200).json({ received: true });
    }

    // 処理済みとして記録
    await prisma.webhookEvent.create({
      data: { id: `stripe-${event.id}`, processedAt: new Date() },
    });

    // 素早く200を返してから処理
    res.status(200).json({ received: true });

    // 重い処理はキューに投げる
    await webhookQueue.add(event.type, event);
  }
);
```

```typescript
// src/webhooks/handlers/paymentHandler.ts
export async function handlePaymentIntentSucceeded(
  event: Stripe.Event
): Promise<void> {
  const paymentIntent = event.data.object as Stripe.PaymentIntent;
  const orderId = paymentIntent.metadata.orderId;

  if (!orderId) {
    logger.warn({ paymentIntentId: paymentIntent.id }, 'No orderId in metadata');
    return;
  }

  await prisma.order.update({
    where: { id: orderId },
    data: {
      status: 'completed',
      paidAt: new Date(),
      paymentIntentId: paymentIntent.id,
    },
  });

  // 完了通知をキューに追加
  await emailQueue.add('order-confirmation', { orderId });
  logger.info({ orderId, paymentIntentId: paymentIntent.id }, 'Order completed');
}
```

---

## GitHub Webhookの検証

```
GitHub WebhookのHMAC-SHA256署名検証を実装してください。

要件：
- X-Hub-Signature-256ヘッダーで検証
- timing-safe比較（タイミング攻撃防止）
- 対象イベント: push, pull_request
- CIパイプラインのトリガーに使う

保存先: src/webhooks/githubWebhook.ts
```

```typescript
// src/webhooks/githubWebhook.ts
import crypto from 'crypto';

function verifyGithubSignature(body: Buffer, signature: string): boolean {
  const hmac = crypto.createHmac('sha256', process.env.GITHUB_WEBHOOK_SECRET!);
  hmac.update(body);
  const expectedSignature = `sha256=${hmac.digest('hex')}`;

  // timing-safe比較（タイミング攻撃防止）
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}

githubWebhookRouter.post(
  '/webhooks/github',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    const signature = req.headers['x-hub-signature-256'] as string;

    if (!signature || !verifyGithubSignature(req.body, signature)) {
      return res.status(401).send('Invalid signature');
    }

    const event = req.headers['x-github-event'] as string;
    const payload = JSON.parse(req.body.toString());

    res.status(200).json({ received: true });

    // イベントタイプに応じて処理
    await githubQueue.add(event, payload);
  }
);
```

---

## まとめ

Claude CodeでWebhook受信を設計する：

1. **CLAUDE.md** に署名検証必須・冪等性必須・素早いレスポンスを明記
2. **rawBodyで署名検証** （JSONパース後では検証できない）
3. **冪等性テーブル** で二重処理を防ぐ
4. **BullMQに投げてから200を返す** （重い処理をブロックしない）

---

*Webhookのセキュリティ問題を自動検出するスキルは **Security Pack（¥1,480）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
