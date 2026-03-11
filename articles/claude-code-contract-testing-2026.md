---
title: "Claude Codeでコントラクトテストを設計する：Pact・API互換性・Consumer-Driven"
emoji: "📝"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "testing", "microservices"]
published: true
---

## はじめに

マイクロサービスでAPIを変更したとき、コンシューマー（呼び出し側）が壊れていないかを検知したい。コントラクトテストでプロバイダー（API提供側）の変更がコンシューマーに影響しないことを自動検証する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにコントラクトテスト設計ルールを書く

```markdown
## コントラクトテスト設計ルール

### Pact（Consumer-Driven Contract Testing）
- コンシューマー側がコントラクト（期待するAPIの形）を定義
- プロバイダー側がコントラクトを満たすか検証
- PactBrokerでコントラクトを共有・バージョン管理

### テスト対象
- レスポンスフィールドの存在確認
- レスポンスのデータ型確認
- HTTPステータスコード確認
- 正確な値は検証しない（環境依存を避ける）

### CI/CDへの統合
- コンシューマー: PR時にコントラクトを生成・PactBrokerにパブリッシュ
- プロバイダー: デプロイ前にコントラクト検証
- can-i-deploy チェック: 安全にデプロイできるか確認
```

---

## コントラクトテストの生成

```
OrderServiceとBFFのコントラクトテストを設計してください。

コンシューマー: BFF（dashboardBff.ts）
プロバイダー: OrderService
要件：
- Pactでコントラクト定義
- プロバイダー検証
- PactBrokerへのパブリッシュ
- GitHub Actionsへの統合

生成ファイル: tests/contract/
```

---

## 生成されるコントラクトテスト

```typescript
// tests/contract/consumer/orderService.pact.ts（BFF側が定義）
import { Pact, Matchers } from '@pact-foundation/pact';
import { OrderServiceClient } from '../../src/clients/orderServiceClient';

const { like, eachLike, integer, string, iso8601DateTimeWithMillis } = Matchers;

const provider = new Pact({
  consumer: 'DashboardBFF',
  provider: 'OrderService',
  port: 8080,
  dir: './pacts', // コントラクトJSON出力先
  logLevel: 'error',
});

describe('OrderService Contract', () => {
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());

  describe('GET /orders?userId=:userId&limit=5', () => {
    test('returns recent orders for a user', async () => {
      // コントラクト定義: BFFが期待するAPIの形
      await provider.addInteraction({
        state: 'user has 3 orders',
        uponReceiving: 'a request for recent orders',
        withRequest: {
          method: 'GET',
          path: '/orders',
          query: { userId: 'user-123', limit: '5' },
          headers: { 'X-Internal-Token': like('some-token') },
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            orders: eachLike({
              id: string('order-id-1'),
              userId: string('user-123'),
              status: string('pending'),
              total: integer(5000),       // 正確な値ではなく「整数であること」を検証
              createdAt: iso8601DateTimeWithMillis('2025-01-01T00:00:00.000Z'),
            }),
          },
        },
      });

      // 実際にクライアントコードを呼び出す（Pactがモックサーバーを起動）
      const client = new OrderServiceClient(provider.mockService.baseUrl);
      const result = await client.getRecentOrders('user-123', 5);

      expect(result).toHaveLength(1); // eachLike は最低1件
      expect(result[0]).toMatchObject({
        id: expect.any(String),
        userId: expect.any(String),
        status: expect.any(String),
        total: expect.any(Number),
      });
    });

    test('returns 404 when user has no orders', async () => {
      await provider.addInteraction({
        state: 'user has no orders',
        uponReceiving: 'a request for orders of user with no orders',
        withRequest: {
          method: 'GET',
          path: '/orders',
          query: { userId: 'user-no-orders', limit: '5' },
          headers: { 'X-Internal-Token': like('some-token') },
        },
        willRespondWith: {
          status: 200,
          body: { orders: [] }, // 空配列を返す（404ではなく）
        },
      });

      const client = new OrderServiceClient(provider.mockService.baseUrl);
      const result = await client.getRecentOrders('user-no-orders', 5);
      expect(result).toEqual([]);
    });
  });
});
```

```typescript
// tests/contract/provider/orderService.verification.ts（OrderService側が検証）
import { Verifier } from '@pact-foundation/pact';
import { createApp } from '../../src/app';

describe('OrderService Provider Verification', () => {
  let server: ReturnType<typeof createApp>;

  beforeAll(async () => {
    server = await createApp();
    await server.listen(8080);
  });

  afterAll(async () => {
    await server.close();
  });

  test('verify pacts from PactBroker', async () => {
    await new Verifier({
      provider: 'OrderService',
      providerBaseUrl: 'http://localhost:8080',

      // PactBrokerからコントラクトを取得
      pactBrokerUrl: process.env.PACT_BROKER_URL!,
      pactBrokerToken: process.env.PACT_BROKER_TOKEN!,

      // Provider Stateの設定（テストデータのセットアップ）
      stateHandlers: {
        'user has 3 orders': async () => {
          await prisma.order.createMany({
            data: [
              { id: 'order-1', userId: 'user-123', status: 'pending', total: 5000 },
              { id: 'order-2', userId: 'user-123', status: 'confirmed', total: 3000 },
              { id: 'order-3', userId: 'user-123', status: 'shipped', total: 8000 },
            ],
          });
        },
        'user has no orders': async () => {
          // データなし（何もしない）
        },
      },

      // デプロイ可能かチェック
      publishVerificationResult: true,
      providerVersion: process.env.APP_VERSION ?? '1.0.0',
    }).verifyProvider();
  });
});
```

---

## GitHub Actionsへの統合

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  consumer-contract:
    name: Generate and publish contracts (BFF)
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - run: npm ci
      - name: Run consumer contract tests
        run: npm run test:contract:consumer
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
      - name: Publish pacts to PactBroker
        run: |
          npx pact-broker publish ./pacts \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --broker-token ${{ secrets.PACT_BROKER_TOKEN }} \
            --consumer-app-version ${{ github.sha }}

  provider-verification:
    name: Verify contracts (OrderService)
    runs-on: ubuntu-latest
    needs: [consumer-contract]

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - name: Verify provider against pacts
        run: npm run test:contract:provider
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          APP_VERSION: ${{ github.sha }}

  can-i-deploy:
    name: Check can-i-deploy
    runs-on: ubuntu-latest
    needs: [provider-verification]

    steps:
      - name: Check if safe to deploy
        run: |
          npx pact-broker can-i-deploy \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --broker-token ${{ secrets.PACT_BROKER_TOKEN }} \
            --pacticipant OrderService \
            --version ${{ github.sha }} \
            --to production
```

---

## まとめ

Claude Codeでコントラクトテストを設計する：

1. **CLAUDE.md** にConsumer-Driven・PactBroker共有・CI/CD統合を明記
2. **コンシューマー側がコントラクトを定義** （プロバイダーへの期待を明文化）
3. **like()/eachLike()** で正確な値ではなく型・構造を検証（環境依存を回避）
4. **can-i-deploy** でデプロイ前に全コンシューマーへの影響を確認

---

*コントラクトテスト設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
