---
title: "Claude CodeでStripe決済を設計する：サブスクリプション・支払いフロー・Webhook連携"
emoji: "💳"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "stripe", "payments"]
published: true
---

## はじめに

Stripe実装は「チェックアウト画面を作る」より難しい——サブスクリプションのライフサイクル管理、Webhookの冪等処理、支払い失敗のリトライ。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにStripe設計ルールを書く

```markdown
## Stripe設計ルール

### セキュリティ（必須）
- サーバーサイドで価格・商品を検証（クライアントの金額を信用しない）
- Webhookは必ず署名検証（rawBodyで検証）
- Stripe Secret KeyはサーバーのみでPublishable Keyのみクライアントに渡す

### サブスクリプション管理
- StripeのcustomerIdをユーザーDBに保存（Stripeで顧客を一意識別）
- subscriptionIdもDBに保存（ステータス管理のため）
- Webhookで subscription.created / customer.subscription.updated / deleted を処理

### 支払いフロー
- Checkout Session: 推奨（Stripeがカード入力フォームを処理）
- Payment Element: カスタムUIが必要な場合
- Priceはハードコードせず環境変数のPrice IDを使う
```

---

## Checkout Sessionの生成

```
Stripeサブスクリプション決済を設計してください。

フロー:
1. POST /api/checkout → Checkout Sessionを作成 → URLを返す
2. ユーザーがStripeのページで支払い
3. Webhook: checkout.session.completed → DB更新
4. Webhook: customer.subscription.deleted → サブスク停止

要件：
- customerをStripeに作成してDBに保存
- Checkout Session でPriceId指定
- Webhookの冪等性処理
```

---

## 生成されるStripe決済

```typescript
// src/services/stripeService.ts
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function getOrCreateStripeCustomer(userId: string): Promise<string> {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  if (user?.stripeCustomerId) {
    return user.stripeCustomerId;
  }

  // Stripeに顧客を作成
  const customer = await stripe.customers.create({
    email: user!.email,
    name: user!.name,
    metadata: { userId }, // StripeからuserIdを追跡できるようにする
  });

  // DBに保存
  await prisma.user.update({
    where: { id: userId },
    data: { stripeCustomerId: customer.id },
  });

  return customer.id;
}

export async function createCheckoutSession(
  userId: string,
  priceId: string
): Promise<string> {
  const customerId = await getOrCreateStripeCustomer(userId);

  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    payment_method_types: ['card'],
    line_items: [{ price: priceId, quantity: 1 }],
    mode: 'subscription',
    success_url: `${process.env.APP_URL}/dashboard?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${process.env.APP_URL}/pricing`,
    subscription_data: {
      metadata: { userId }, // Webhookでユーザーを特定するために
    },
    // トライアル期間
    // subscription_data: { trial_period_days: 14 }
  });

  return session.url!;
}
```

```typescript
// src/routes/checkout.ts
router.post('/api/checkout', authenticate, async (req, res) => {
  // Price IDはクライアントから受け取るが、サーバーで検証
  const { planId } = req.body;

  const VALID_PLANS = {
    basic: process.env.STRIPE_BASIC_PRICE_ID!,
    pro: process.env.STRIPE_PRO_PRICE_ID!,
  } as const;

  const priceId = VALID_PLANS[planId as keyof typeof VALID_PLANS];
  if (!priceId) {
    return res.status(400).json({ error: 'Invalid plan' });
  }

  const checkoutUrl = await createCheckoutSession(req.user.id, priceId);
  res.json({ url: checkoutUrl });
});
```

---

## Webhook処理

```typescript
// src/webhooks/stripeWebhook.ts
stripeWebhookRouter.post(
  '/webhooks/stripe',
  express.raw({ type: 'application/json' }),
  async (req, res) => {
    const sig = req.headers['stripe-signature'] as string;
    let event: Stripe.Event;

    try {
      event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
    } catch (err) {
      return res.status(400).send('Signature verification failed');
    }

    // 冪等性チェック
    const already = await prisma.webhookEvent.findUnique({ where: { id: `stripe-${event.id}` } });
    if (already) return res.json({ received: true });

    await prisma.webhookEvent.create({
      data: { id: `stripe-${event.id}`, type: event.type, processedAt: new Date() },
    });

    res.json({ received: true }); // 即時200を返す

    // 非同期処理
    switch (event.type) {
      case 'checkout.session.completed': {
        const session = event.data.object as Stripe.CheckoutSession;
        const userId = session.subscription_data?.metadata?.userId ?? session.metadata?.userId;

        if (userId && session.subscription) {
          await prisma.user.update({
            where: { id: userId },
            data: {
              subscriptionId: session.subscription as string,
              subscriptionStatus: 'active',
              plan: getPlanFromPriceId(session),
            },
          });
        }
        break;
      }

      case 'customer.subscription.deleted': {
        const subscription = event.data.object as Stripe.Subscription;
        await prisma.user.updateMany({
          where: { subscriptionId: subscription.id },
          data: { subscriptionStatus: 'cancelled', plan: 'free' },
        });
        break;
      }
    }
  }
);
```

---

## まとめ

Claude CodeでStripe決済を設計する：

1. **CLAUDE.md** に価格サーバー検証必須・Webhook署名検証必須・Secret Keyはサーバーのみを明記
2. **getOrCreateStripeCustomer** でStripeの顧客とDB userを紐付け
3. **Checkout Session** でカード入力をStripeに任せる（PCI準拠）
4. **Webhook + 冪等性テーブル** で二重処理を防ぐ

---

*Stripe実装のセキュリティレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
