---
title: "Claude Codeでテスト戦略を設計する：ユニット・統合・E2E・テストピラミッド"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "testing", "vitest"]
published: true
---

## はじめに

「テストはあるが、何をどこまでテストするか分からない」——テストピラミッドで戦略を決め、コストパフォーマンスの高いテスト構成を作る。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにテスト戦略を書く

```markdown
## テスト戦略ルール

### テストピラミッド
- ユニットテスト（70%）: ビジネスロジック、バリデーション、純粋関数
- 統合テスト（20%）: DBアクセス、外部API、ミドルウェア
- E2Eテスト（10%）: クリティカルパスのみ（ログイン→注文→決済）

### テストルール
- DBを使う統合テストはTransactionで各テストを分離（ロールバック）
- 外部APIはMSWでモック（実際のAPIに依存しない）
- テスト名は「〜のとき、〜すべき」形式（仕様ドキュメントとして機能）

### カバレッジ目標
- ユニットテスト: ビジネスロジック100%
- 統合テスト: APIエンドポイント80%以上
- E2Eテスト: クリティカルパス100%
```

---

## テスト設計の生成

```
OrderServiceの包括的なテストを設計してください。

対象: createOrder関数（在庫確認→決済→DB保存）
要件：
- ユニットテスト（Vitest）
- 統合テスト（実DB、Transaction分離）
- MSWでStripe APIモック

生成ファイル: tests/
```

---

## 生成されるユニットテスト

```typescript
// tests/unit/services/orderService.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { createOrder, calculateOrderTotal } from '../../../src/services/orderService';
import { ConflictError, ValidationError } from '../../../src/errors';

// 依存関係をモック
vi.mock('../../../src/lib/prisma', () => ({
  prisma: {
    order: { create: vi.fn() },
    inventory: { findMany: vi.fn() },
  },
}));

vi.mock('../../../src/lib/stripe', () => ({
  stripe: {
    paymentIntents: { create: vi.fn() },
  },
}));

describe('createOrder', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('有効な入力で注文が作成できるべき', async () => {
    // Arrange
    const mockInventory = [{ productId: 'prod-1', quantity: 10 }];
    vi.mocked(prisma.inventory.findMany).mockResolvedValue(mockInventory);
    vi.mocked(stripe.paymentIntents.create).mockResolvedValue({ id: 'pi_test123' } as any);
    vi.mocked(prisma.order.create).mockResolvedValue({
      id: 'order-1', userId: 'user-1', total: 5000, status: 'pending',
    } as any);

    // Act
    const result = await createOrder('user-1', [{
      productId: 'prod-1', quantity: 2, priceCents: 2500,
    }]);

    // Assert
    expect(result.id).toBe('order-1');
    expect(result.status).toBe('pending');
    expect(prisma.order.create).toHaveBeenCalledOnce();
    expect(stripe.paymentIntents.create).toHaveBeenCalledWith(
      expect.objectContaining({ amount: 5000, currency: 'jpy' })
    );
  });

  it('在庫不足のとき ConflictError をスローすべき', async () => {
    vi.mocked(prisma.inventory.findMany).mockResolvedValue([
      { productId: 'prod-1', quantity: 1 }, // 在庫1個
    ]);

    await expect(
      createOrder('user-1', [{ productId: 'prod-1', quantity: 5 }]) // 5個注文
    ).rejects.toThrow(ConflictError);

    expect(stripe.paymentIntents.create).not.toHaveBeenCalled();
    expect(prisma.order.create).not.toHaveBeenCalled();
  });

  it('空のitemsで ValidationError をスローすべき', async () => {
    await expect(createOrder('user-1', [])).rejects.toThrow(ValidationError);
  });
});

describe('calculateOrderTotal', () => {
  it('複数アイテムの合計を正しく計算すべき', () => {
    const items = [
      { priceCents: 1000, quantity: 2 },
      { priceCents: 500, quantity: 3 },
    ];
    expect(calculateOrderTotal(items)).toBe(3500);
  });

  it('空のアイテムリストは0を返すべき', () => {
    expect(calculateOrderTotal([])).toBe(0);
  });
});
```

---

## 統合テスト（実DB + Transaction分離）

```typescript
// tests/integration/api/orders.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { createApp } from '../../../src/app';
import { prisma } from '../../../src/lib/prisma';
import request from 'supertest';
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

// MSWでStripe APIをモック
const mswServer = setupServer(
  http.post('https://api.stripe.com/v1/payment_intents', () => {
    return HttpResponse.json({ id: 'pi_test_mock', status: 'requires_payment_method' });
  })
);

let app: Express;

beforeAll(async () => {
  mswServer.listen({ onUnhandledRequest: 'error' });
  app = await createApp();
});

afterAll(async () => {
  mswServer.close();
  await prisma.$disconnect();
});

beforeEach(async () => {
  // トランザクションで各テストを分離（テスト後にロールバック）
  // ※ Vitestのbeforeメソッドではbeginできないため、テスト内でtransactionを使う
  await prisma.order.deleteMany(); // 簡単のため全削除
  await prisma.inventory.upsert({
    where: { productId: 'test-product-1' },
    create: { productId: 'test-product-1', quantity: 10 },
    update: { quantity: 10 },
  });
});

describe('POST /api/v1/orders', () => {
  it('認証済みユーザーが有効なリクエストで注文を作成できる', async () => {
    const jwtToken = generateTestJwt('test-user-1');

    const response = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${jwtToken}`)
      .set('Idempotency-Key', '550e8400-e29b-41d4-a716-446655440000')
      .send({
        items: [{ productId: 'test-product-1', quantity: 2, priceCents: 2500 }],
      });

    expect(response.status).toBe(201);
    expect(response.body).toMatchObject({
      id: expect.any(String),
      userId: 'test-user-1',
      status: 'pending',
      total: 5000,
    });

    // DBに実際に保存されているか確認
    const savedOrder = await prisma.order.findUnique({
      where: { id: response.body.id },
    });
    expect(savedOrder).toBeTruthy();
  });

  it('未認証の場合は 401 を返す', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .send({ items: [{ productId: 'test-product-1', quantity: 1 }] });

    expect(response.status).toBe(401);
  });

  it('Idempotency-Keyが同じリクエストは同じレスポンスを返す', async () => {
    const jwtToken = generateTestJwt('test-user-2');
    const idempotencyKey = '550e8400-e29b-41d4-a716-446655440001';
    const requestBody = { items: [{ productId: 'test-product-1', quantity: 1 }] };

    const first = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${jwtToken}`)
      .set('Idempotency-Key', idempotencyKey)
      .send(requestBody);

    const second = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${jwtToken}`)
      .set('Idempotency-Key', idempotencyKey)
      .send(requestBody);

    expect(first.status).toBe(201);
    expect(second.status).toBe(201);
    expect(second.headers['idempotency-replayed']).toBe('true');
    expect(second.body.id).toBe(first.body.id); // 同じ注文IDが返る
  });
});
```

---

## テスト設定（Vitest）

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 70,
      },
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.d.ts', 'src/types/**'],
    },
    // 統合テストは別プロジェクトで管理
    projects: [
      { test: { name: 'unit', include: ['tests/unit/**/*.test.ts'] } },
      {
        test: {
          name: 'integration',
          include: ['tests/integration/**/*.test.ts'],
          setupFiles: ['./tests/integration-setup.ts'],
        },
      },
    ],
  },
});
```

---

## まとめ

Claude Codeでテスト戦略を設計する：

1. **CLAUDE.md** にテストピラミッド比率・Transaction分離・MSWモックを明記
2. **ユニットテスト** でビジネスロジックを高速検証（vi.mock()で依存を切り離す）
3. **統合テスト** で実DBとAPIを一気通貫テスト（MSWで外部APIはモック）
4. **Idempotency**・**認証**・**バリデーション** を統合テストでカバー

---

*テスト設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
