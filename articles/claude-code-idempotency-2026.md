---
title: "Claude Codeで冪等性を設計する：重複リクエスト・リトライセーフ・Idempotency Key"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "api"]
published: true
---

## はじめに

ネットワーク障害でクライアントがリトライすると、同じ注文が二重に作成される。Idempotency Keyで同じリクエストは同じレスポンスを返し、副作用を1回だけ実行させる。Claude Codeに設計を生成させる。

---

## CLAUDE.mdに冪等性設計ルールを書く

```markdown
## 冪等性設計ルール

### Idempotency Key
- POSTリクエストには必ずIdempotency-Keyヘッダーを要求（金融・注文）
- キーの有効期限: 24時間（それ以降は新規処理）
- レスポンスはキャッシュして同じレスポンスを返す
- 処理中（IN_FLIGHT）のリクエストは409で返す（重複実行防止）

### 冪等な操作の設計
- DB: INSERT ... ON CONFLICT DO NOTHING（重複挿入を無視）
- 外部API: 外部APIのIdempotency Keyを自分のキーから導出
- メッセージキュー: jobIdを固定値にして重複キューイングを防ぐ

### リトライセーフな操作
- GET/PUT/DELETE: 本質的に冪等
- POST: Idempotency Keyで冪等にする
- 外部課金（Stripe等）: 外部のIdempotency Keyを必ず使う
```

---

## 冪等性システムの生成

```
API全体の冪等性システムを設計してください。

要件：
- Idempotency-Key ヘッダー検証ミドルウェア
- Redisでリクエスト/レスポンスキャッシュ
- 処理中の重複リクエストを409で返す
- Stripe Checkoutへの適用例

生成ファイル: src/middleware/idempotency.ts
```

---

## 生成される冪等性実装

```typescript
// src/middleware/idempotency.ts

const IDEMPOTENCY_TTL = 86400; // 24時間

interface IdempotencyRecord {
  status: 'in_flight' | 'completed';
  statusCode?: number;
  responseBody?: unknown;
  createdAt: string;
}

export function idempotencyMiddleware(options: { required?: boolean } = {}) {
  return async (req: Request, res: Response, next: NextFunction) => {
    // GETなど冪等なメソッドはスキップ
    if (['GET', 'HEAD', 'OPTIONS', 'DELETE', 'PUT'].includes(req.method)) {
      return next();
    }

    const idempotencyKey = req.headers['idempotency-key'] as string | undefined;

    if (!idempotencyKey) {
      if (options.required) {
        return res.status(400).json({
          error: 'Idempotency-Key header is required for this endpoint',
        });
      }
      return next();
    }

    // キーのフォーマット検証（UUIDのみ受け付ける）
    if (!/^[0-9a-f-]{36}$/i.test(idempotencyKey)) {
      return res.status(400).json({ error: 'Invalid Idempotency-Key format (UUID required)' });
    }

    const cacheKey = `idempotency:${req.user?.id}:${idempotencyKey}`;

    // 既存のレコードを確認
    const existing = await redis.get(cacheKey);

    if (existing) {
      const record: IdempotencyRecord = JSON.parse(existing);

      if (record.status === 'in_flight') {
        // 同じリクエストが処理中 → 409 Conflict
        return res.status(409).json({
          error: 'A request with this Idempotency-Key is already being processed',
          idempotencyKey,
        });
      }

      if (record.status === 'completed') {
        // 完了済み → キャッシュされたレスポンスを返す
        logger.info({ idempotencyKey, cacheKey }, 'Idempotency cache hit');
        res.setHeader('Idempotency-Replayed', 'true');
        return res.status(record.statusCode!).json(record.responseBody);
      }
    }

    // IN_FLIGHTとしてマーク（処理中の重複を防ぐ）
    const inFlightRecord: IdempotencyRecord = {
      status: 'in_flight',
      createdAt: new Date().toISOString(),
    };
    await redis.set(cacheKey, JSON.stringify(inFlightRecord), { EX: 300 }); // 5分でタイムアウト

    // レスポンスをインターセプトしてキャッシュ
    const originalJson = res.json.bind(res);
    res.json = (body: unknown) => {
      const completedRecord: IdempotencyRecord = {
        status: 'completed',
        statusCode: res.statusCode,
        responseBody: body,
        createdAt: new Date().toISOString(),
      };

      // 成功レスポンス（2xx）のみキャッシュ
      if (res.statusCode >= 200 && res.statusCode < 300) {
        redis.set(cacheKey, JSON.stringify(completedRecord), { EX: IDEMPOTENCY_TTL })
          .catch(err => logger.error({ err }, 'Idempotency cache save failed'));
      } else {
        // エラーの場合はIN_FLIGHTレコードを削除（リトライ可能にする）
        redis.del(cacheKey).catch(err => logger.error({ err }, 'Idempotency cache delete failed'));
      }

      return originalJson(body);
    };

    next();
  };
}
```

---

## エンドポイントへの適用

```typescript
// 注文作成: Idempotency-Key必須
router.post(
  '/orders',
  authenticate,
  idempotencyMiddleware({ required: true }),
  async (req, res) => {
    const order = await createOrder(req.user.id, req.body);
    res.status(201).json(order);
  }
);

// 支払いリトライ: Idempotency-Key必須
router.post(
  '/payments',
  authenticate,
  idempotencyMiddleware({ required: true }),
  async (req, res) => {
    const payment = await processPayment(req.user.id, req.body);
    res.status(201).json(payment);
  }
);
```

---

## Stripe外部APIへの冪等性適用

```typescript
// src/services/stripeService.ts

export async function createCheckoutSessionIdempotent(
  userId: string,
  items: CartItem[],
  clientIdempotencyKey: string // クライアントから受け取ったキー
): Promise<Stripe.Checkout.Session> {
  // クライアントのキーからStripe用のキーを導出
  // （ユーザーIDを含めてキーのなりすましを防ぐ）
  const stripeIdempotencyKey = `checkout:${userId}:${clientIdempotencyKey}`;

  const session = await stripe.checkout.sessions.create(
    {
      payment_method_types: ['card'],
      line_items: items.map(item => ({
        price_data: {
          currency: 'jpy',
          product_data: { name: item.name },
          unit_amount: item.priceCents,
        },
        quantity: item.quantity,
      })),
      mode: 'payment',
      success_url: `${process.env.APP_URL}/checkout/success`,
      cancel_url: `${process.env.APP_URL}/checkout/cancel`,
    },
    {
      idempotencyKey: stripeIdempotencyKey, // Stripeの冪等性キー
    }
  );

  return session;
}
```

---

## DBレベルの冪等性（INSERT ... ON CONFLICT）

```typescript
// 外部IDをユニーク制約で冪等性保証
async function createOrderIdempotent(
  externalOrderId: string, // クライアント生成のユニークID
  data: CreateOrderData
): Promise<Order> {
  try {
    return await prisma.order.create({
      data: {
        externalId: externalOrderId, // UNIQUE制約
        ...data,
      },
    });
  } catch (err) {
    // P2002: ユニーク制約違反 = 既に存在する
    if (isPrismaError(err) && err.code === 'P2002') {
      // 既存のレコードを返す（冪等）
      return prisma.order.findUniqueOrThrow({
        where: { externalId: externalOrderId },
      });
    }
    throw err;
  }
}
```

---

## まとめ

Claude Codeで冪等性を設計する：

1. **CLAUDE.md** にIdempotency-Key必須エンドポイント・TTL 24時間・IN_FLIGHT 409を明記
2. **IN_FLIGHTマーク** で処理中の重複リクエストを即座に409返却（二重実行防止）
3. **成功レスポンスのみキャッシュ** してエラー時はリトライ可能にする
4. **外部API（Stripe）** にも自分のキーから導出したIdempotency Keyを渡す

---

*冪等性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
