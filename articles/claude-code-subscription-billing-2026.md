---
title: "Claude Codeでサブスクリプション課金を設計する：Stripe Billing・プラン変更・Webhook"
emoji: "💳"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "stripe", "saas"]
published: true
---

## はじめに

SaaSのサブスクリプション課金——月次/年次プラン、プランアップグレード、解約、支払い失敗時のアクセス制限を実装する。Stripe Billingを使いClaude Codeに設計させる。

---

## CLAUDE.mdにサブスクリプション課金設計ルールを書く

```markdown
## サブスクリプション課金設計ルール

### Stripe Billing設計
- Customer/Subscription/ProductはStripeが正
- ローカルDBはStripeのWebhookで同期
- 手動でDBを更新しない（Webhookが唯一の真実）

### プラン管理
- プランはStripe ProductとPriceで定義
- アップグレード: 即時反映（日割り計算自動）
- ダウングレード: 次の請求サイクルから反映
- 解約: 期間終了まで継続利用可

### 支払い失敗
- 失敗から3日後・7日後・14日後にリトライ
- 最終失敗でサブスク停止→アクセス制限
- アクセス制限はソフトブロック（閲覧可・作成不可）

### Webhook
- 全Webhookはidempotency保証
- 署名検証必須（stripe-signature）
- 処理失敗は必ずエラーログ＋Slack通知
```

---

## サブスクリプション課金の生成

```
Stripe Billingサブスクリプション管理を設計してください。

プラン: Free / Pro（¥1,980/月） / Business（¥9,800/月）
要件：
- サブスクリプション作成・変更・解約
- 支払い失敗時のアクセス制限
- Webhookでのイベント処理

生成ファイル: src/billing/
```

---

## 生成されるサブスクリプション課金実装

```typescript
// src/billing/stripeService.ts

// プラン定義（Stripe Price IDで管理）
const PLANS = {
  free: null, // Stripeサブスクリプション不要
  pro: {
    monthly: process.env.STRIPE_PRICE_PRO_MONTHLY!,   // price_xxx
    annual: process.env.STRIPE_PRICE_PRO_ANNUAL!,
  },
  business: {
    monthly: process.env.STRIPE_PRICE_BUSINESS_MONTHLY!,
    annual: process.env.STRIPE_PRICE_BUSINESS_ANNUAL!,
  },
} as const;

// サブスクリプション作成
export async function createSubscription(
  userId: string,
  plan: 'pro' | 'business',
  billingCycle: 'monthly' | 'annual',
  paymentMethodId: string
): Promise<Stripe.Subscription> {
  const user = await prisma.user.findUniqueOrThrow({ where: { id: userId } });

  // Stripe Customerを取得または作成
  let customerId = user.stripeCustomerId;
  if (!customerId) {
    const customer = await stripe.customers.create({
      email: user.email,
      name: user.name,
      metadata: { userId },
    });
    customerId = customer.id;
    await prisma.user.update({
      where: { id: userId },
      data: { stripeCustomerId: customerId },
    });
  }

  // デフォルト支払い方法を設定
  await stripe.customers.update(customerId, {
    invoice_settings: { default_payment_method: paymentMethodId },
  });

  const priceId = PLANS[plan][billingCycle];

  // サブスクリプション作成
  const subscription = await stripe.subscriptions.create({
    customer: customerId,
    items: [{ price: priceId }],
    payment_settings: {
      payment_method_types: ['card'],
      save_default_payment_method: 'on_subscription',
    },
    expand: ['latest_invoice.payment_intent'],
  });

  // DBに即時反映（Webhookが届くまでの間もアクセスできるよう）
  await prisma.subscription.upsert({
    where: { userId },
    create: {
      userId,
      stripeSubscriptionId: subscription.id,
      stripeCustomerId: customerId,
      plan,
      status: subscription.status,
      currentPeriodEnd: new Date(subscription.current_period_end * 1000),
    },
    update: {
      stripeSubscriptionId: subscription.id,
      plan,
      status: subscription.status,
      currentPeriodEnd: new Date(subscription.current_period_end * 1000),
    },
  });

  return subscription;
}

// プランアップグレード（即時反映）
export async function upgradeSubscription(
  userId: string,
  newPlan: 'pro' | 'business',
  billingCycle: 'monthly' | 'annual'
): Promise<void> {
  const sub = await prisma.subscription.findUniqueOrThrow({ where: { userId } });
  const stripeSub = await stripe.subscriptions.retrieve(sub.stripeSubscriptionId);

  const newPriceId = PLANS[newPlan][billingCycle];

  await stripe.subscriptions.update(sub.stripeSubscriptionId, {
    items: [{
      id: stripeSub.items.data[0].id,
      price: newPriceId,
    }],
    proration_behavior: 'always_invoice', // 差額を即時請求（日割り計算）
  });

  // Webhookが処理するまでの間もUI上でプランが更新されるよう楽観的更新
  await prisma.subscription.update({
    where: { userId },
    data: { plan: newPlan, updatedAt: new Date() },
  });
}

// サブスクリプション解約
export async function cancelSubscription(
  userId: string,
  reason?: string
): Promise<void> {
  const sub = await prisma.subscription.findUniqueOrThrow({ where: { userId } });

  // 期間終了時に解約（即時解約はしない）
  await stripe.subscriptions.update(sub.stripeSubscriptionId, {
    cancel_at_period_end: true,
    metadata: { cancellationReason: reason ?? 'user_requested' },
  });

  await prisma.subscription.update({
    where: { userId },
    data: { cancelAtPeriodEnd: true },
  });
}
```

---

## Webhookイベント処理

```typescript
// src/billing/webhookHandler.ts

export async function handleStripeWebhook(
  payload: Buffer,
  signature: string
): Promise<void> {
  // 署名検証
  const event = stripe.webhooks.constructEvent(
    payload,
    signature,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  // 冪等性: 同じイベントを2回処理しない
  const existing = await prisma.processedWebhook.findUnique({
    where: { stripeEventId: event.id },
  });
  if (existing) {
    logger.info({ eventId: event.id }, 'Webhook already processed');
    return;
  }

  switch (event.type) {
    case 'invoice.payment_succeeded': {
      const invoice = event.data.object as Stripe.Invoice;
      await handlePaymentSucceeded(invoice);
      break;
    }

    case 'invoice.payment_failed': {
      const invoice = event.data.object as Stripe.Invoice;
      await handlePaymentFailed(invoice);
      break;
    }

    case 'customer.subscription.updated': {
      const sub = event.data.object as Stripe.Subscription;
      await syncSubscription(sub);
      break;
    }

    case 'customer.subscription.deleted': {
      const sub = event.data.object as Stripe.Subscription;
      await handleSubscriptionCancelled(sub);
      break;
    }
  }

  // 処理済みとして記録
  await prisma.processedWebhook.create({
    data: { stripeEventId: event.id, type: event.type, processedAt: new Date() },
  });
}

async function handlePaymentFailed(invoice: Stripe.Invoice): Promise<void> {
  const customerId = invoice.customer as string;
  const sub = await prisma.subscription.findFirst({
    where: { stripeCustomerId: customerId },
  });
  if (!sub) return;

  const attemptCount = invoice.attempt_count ?? 1;

  if (attemptCount >= 3) {
    // 最終失敗: アクセス制限
    await prisma.subscription.update({
      where: { id: sub.id },
      data: { status: 'past_due', accessBlocked: true },
    });

    await sendNotification(sub.userId, 'subscription_access_blocked', {
      retryUrl: `${process.env.APP_URL}/billing/retry`,
    });
  } else {
    await prisma.subscription.update({
      where: { id: sub.id },
      data: { status: 'past_due' },
    });

    await sendNotification(sub.userId, 'payment_failed', {
      attemptCount,
      nextRetryDate: new Date(invoice.next_payment_attempt! * 1000),
    });
  }
}
```

---

## まとめ

Claude CodeでStripe Billing課金を設計する：

1. **CLAUDE.md** にStripeが正・Webhookでのみ同期・手動DB更新禁止を明記
2. **アップグレードは即時** (`proration_behavior: 'always_invoice'`で日割り自動計算)
3. **解約は期間終了時** (`cancel_at_period_end: true`)で即時解約しない
4. **Webhookはidempotency** で重複処理を防ぐ（stripeEventIdで管理）

---

*サブスクリプション課金設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
