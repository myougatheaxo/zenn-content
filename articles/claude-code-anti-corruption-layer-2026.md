---
title: "Claude Codeでアンチコラプションレイヤーを設計する：外部システム統合・型変換・ドメインの保護"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-19 09:00"
---

## はじめに

「外部決済APIのモデルがドメインに漏れて、Stripeの概念がビジネスロジックに混在した」——アンチコラプションレイヤー（ACL）で外部システムとの境界を明確にし、外部の概念がドメインを汚染しないよう保護する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにアンチコラプションレイヤー設計ルールを書く

```markdown
## アンチコラプションレイヤー設計ルール

### 役割
- 外部システムのデータモデルをドメインモデルに変換
- 外部APIのインターフェースをドメインに合わせたインターフェースでラップ
- 外部システムの概念（Stripe/Twilio等の用語）がドメイン層に出てこない

### 翻訳
- 外部→内部: Mapper/Translatorで変換
- 内部→外部: Adapterで変換
- エラーも変換: Stripe APIエラー → ドメインエラー

### 依存の方向
- ドメイン層はACLに依存しない（インターフェースのみに依存）
- ACLがインフラ層に位置（ドメイン層より外側）
- テスト: ACLをモック/スタブして差し替え可能
```

---

## ACL実装の生成

```
アンチコラプションレイヤーを設計してください。

要件：
- 決済サービス（Stripe）のACL
- ドメインインターフェース
- エラー変換
- テスト容易性

生成ファイル: src/infrastructure/acl/
```

---

## 生成されるACL実装

```typescript
// src/domain/payment/paymentGateway.ts — ドメインのインターフェース（Stripeを知らない）

export interface PaymentIntent {
  paymentId: string;
  amount: Money;
  status: 'pending' | 'succeeded' | 'failed';
  errorCode?: PaymentErrorCode;
}

export type PaymentErrorCode =
  | 'insufficient_funds'
  | 'card_declined'
  | 'expired_card'
  | 'invalid_cvc'
  | 'processing_error';

export interface IPaymentGateway {
  createPaymentIntent(amount: Money, customerId: string): Promise<PaymentIntent>;
  confirmPayment(paymentId: string): Promise<PaymentIntent>;
  refundPayment(paymentId: string, amount?: Money): Promise<void>;
  getPaymentStatus(paymentId: string): Promise<PaymentIntent>;
}
```

```typescript
// src/infrastructure/acl/stripePaymentGateway.ts — Stripe ACL

import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2023-10-16' });

export class StripePaymentGateway implements IPaymentGateway {
  async createPaymentIntent(amount: Money, customerId: string): Promise<PaymentIntent> {
    try {
      // ドメインのMoney → Stripeのamount（通貨の最小単位）
      const stripeIntent = await stripe.paymentIntents.create({
        amount: this.toStripeAmount(amount),
        currency: amount.currency.toLowerCase(),
        customer: customerId,  // StripeのcustomerID（内部ではドメインのuserIdと別で管理）
        automatic_payment_methods: { enabled: true },
      });

      return this.toPaymentIntent(stripeIntent);
    } catch (error) {
      throw this.translateError(error);
    }
  }

  async confirmPayment(paymentId: string): Promise<PaymentIntent> {
    try {
      const stripeIntent = await stripe.paymentIntents.confirm(paymentId);
      return this.toPaymentIntent(stripeIntent);
    } catch (error) {
      throw this.translateError(error);
    }
  }

  async refundPayment(paymentId: string, amount?: Money): Promise<void> {
    try {
      await stripe.refunds.create({
        payment_intent: paymentId,
        amount: amount ? this.toStripeAmount(amount) : undefined,
      });
    } catch (error) {
      throw this.translateError(error);
    }
  }

  async getPaymentStatus(paymentId: string): Promise<PaymentIntent> {
    const stripeIntent = await stripe.paymentIntents.retrieve(paymentId);
    return this.toPaymentIntent(stripeIntent);
  }

  // Stripe PaymentIntent → ドメインのPaymentIntent（翻訳）
  private toPaymentIntent(stripe: Stripe.PaymentIntent): PaymentIntent {
    const currency = stripe.currency.toUpperCase() as Currency;

    return {
      paymentId: stripe.id,
      amount: Money.of(stripe.amount, currency),
      status:
        stripe.status === 'succeeded' ? 'succeeded' :
        stripe.status === 'canceled' ? 'failed' :
        'pending',
    };
  }

  // ドメインのMoney → Stripeのamount
  private toStripeAmount(money: Money): number {
    // JPYは1円=1（最小単位）、USD/EURは1セント=100
    return money.currency === 'JPY' ? money.amount : money.amount;
  }

  // StripeエラーをドメインエラーへPayment翻訳
  private translateError(error: unknown): Error {
    if (error instanceof Stripe.errors.StripeCardError) {
      const errorCode = this.mapStripeDeclineCode(error.code ?? '');
      return new PaymentError(errorCode, error.message);
    }

    if (error instanceof Stripe.errors.StripeInvalidRequestError) {
      return new PaymentError('processing_error', 'Invalid payment request');
    }

    if (error instanceof Stripe.errors.StripeAPIError) {
      return new PaymentGatewayUnavailableError('Payment service temporarily unavailable');
    }

    return error as Error;
  }

  private mapStripeDeclineCode(code: string): PaymentErrorCode {
    const mapping: Record<string, PaymentErrorCode> = {
      'insufficient_funds': 'insufficient_funds',
      'card_declined': 'card_declined',
      'expired_card': 'expired_card',
      'incorrect_cvc': 'invalid_cvc',
      'processing_error': 'processing_error',
    };
    return mapping[code] ?? 'processing_error';
  }
}
```

```typescript
// ドメインサービス: IPaymentGatewayを使う（Stripeを知らない）
export class PaymentService {
  constructor(private readonly gateway: IPaymentGateway) {}

  async processPayment(orderId: string, amount: Money, customerId: string): Promise<string> {
    const intent = await this.gateway.createPaymentIntent(amount, customerId);

    if (intent.status === 'failed') {
      throw new PaymentError(intent.errorCode ?? 'processing_error', 'Payment failed');
    }

    logger.info({ orderId, paymentId: intent.paymentId, amount: amount.toString() }, 'Payment intent created');
    return intent.paymentId;
  }
}

// テスト: ACLをスタブに差し替え
class StubPaymentGateway implements IPaymentGateway {
  private _shouldFail = false;

  simulateFailure(): void {
    this._shouldFail = true;
  }

  async createPaymentIntent(amount: Money): Promise<PaymentIntent> {
    if (this._shouldFail) {
      throw new PaymentError('card_declined', 'Card declined (test)');
    }
    return {
      paymentId: `test-${ulid()}`,
      amount,
      status: 'succeeded',
    };
  }

  async confirmPayment(paymentId: string): Promise<PaymentIntent> {
    return { paymentId, amount: Money.zero('JPY'), status: 'succeeded' };
  }

  async refundPayment(): Promise<void> {}
  async getPaymentStatus(paymentId: string): Promise<PaymentIntent> {
    return { paymentId, amount: Money.zero('JPY'), status: 'succeeded' };
  }
}

// 本番: DIコンテナで依存を解決
const paymentService = new PaymentService(new StripePaymentGateway());

// テスト: スタブを注入
const stubGateway = new StubPaymentGateway();
stubGateway.simulateFailure();
const testPaymentService = new PaymentService(stubGateway);
```

---

## まとめ

Claude Codeでアンチコラプションレイヤーを設計する：

1. **CLAUDE.md** にドメインはインターフェースのみに依存（IPaymentGateway）・ACLが外部→内部の変換を担当・Stripeの用語はACL内にのみ出現を明記
2. **`toPaymentIntent()`翻訳メソッド** でStripeの`PaymentIntent.status`をドメインの`'succeeded' | 'pending' | 'failed'`にマッピング——Stripeが`'processing'` → `'requires_confirmation'`と変更してもドメインに影響しない
3. **エラー変換** でStripeCardErrorをPaymentErrorに変換——ドメイン層がStripeのエラークラス階層を知る必要がなく、テスト時も簡単にエラーをシミュレートできる
4. **スタブ実装のテスト容易性** でStripeなしで決済ロジックをテスト——`IPaymentGateway`を実装したスタブを注入するだけで「カード拒否」「決済成功」のシナリオをテスト

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
