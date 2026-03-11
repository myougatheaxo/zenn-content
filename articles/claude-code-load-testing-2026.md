---
title: "Claude Codeで負荷テストを設計する：k6・スループット目標・ボトルネック検出"
emoji: "📊"
type: "tech"
topics: ["claudecode", "k6", "nodejs", "devops", "performance"]
published: true
---

## はじめに

本番でいきなりトラフィックが来て落ちる——事前に負荷テストしていれば防げた。Claude Codeにk6を使った負荷テスト設計を生成させる。

---

## CLAUDE.mdに負荷テストルールを書く

```markdown
## 負荷テスト設計ルール

### テスト種別
- スモークテスト: 最小負荷で動作確認（1 VU、1分）
- 負荷テスト: 期待するトラフィックでSLA確認（目標QPS、5-30分）
- ストレステスト: 限界点を探す（段階的に増加）
- スパイクテスト: 急激な負荷増加への対応（瞬間的に10倍）

### SLA目標（デフォルト）
- P95 レスポンスタイム < 500ms
- P99 レスポンスタイム < 1000ms
- エラーレート < 0.1%
- スループット > 目標QPS

### 測定項目
- VU（仮想ユーザー）数
- レスポンスタイム（P50/P95/P99）
- スループット（requests/sec）
- エラーレート
```

---

## 負荷テストの生成

```
k6で負荷テストスクリプトを生成してください。

ターゲット: POST /api/orders API
テスト種別: 負荷テスト（段階的に50 VUまで増加）

シナリオ:
1. ログインしてJWTを取得
2. 商品一覧を取得
3. 注文を作成

SLA目標:
- P95 < 500ms
- エラーレート < 0.1%

生成ファイル: k6/order-load-test.js
```

---

## 生成されるk6負荷テスト

```javascript
// k6/order-load-test.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Counter, Rate, Trend } from 'k6/metrics';

// カスタムメトリクス
const orderCreationDuration = new Trend('order_creation_duration');
const orderCreationErrors = new Counter('order_creation_errors');
const successRate = new Rate('success_rate');

export const options = {
  stages: [
    { duration: '2m', target: 10 },   // ウォームアップ: 10 VUまで増加
    { duration: '5m', target: 50 },   // メインロード: 50 VUまで増加
    { duration: '3m', target: 50 },   // 維持
    { duration: '2m', target: 0 },    // クールダウン
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // SLA目標
    http_req_failed: ['rate<0.001'],                 // エラーレート < 0.1%
    success_rate: ['rate>0.99'],
    order_creation_duration: ['p(95)<500'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

function authenticate() {
  const res = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: `test-${__VU}@example.com`,
    password: 'TestPassword123!',
  }), {
    headers: { 'Content-Type': 'application/json' },
  });

  check(res, { 'login successful': r => r.status === 200 });
  return res.json('accessToken');
}

export default function () {
  const token = authenticate();
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  };

  group('order flow', () => {
    // Step 1: 商品一覧
    const productsRes = http.get(`${BASE_URL}/api/products?limit=10`, { headers });
    check(productsRes, { 'products loaded': r => r.status === 200 });

    const products = productsRes.json('data');
    if (!products?.length) return;

    // Step 2: 注文作成
    const orderStart = Date.now();
    const orderRes = http.post(
      `${BASE_URL}/api/orders`,
      JSON.stringify({
        items: [{ productId: products[0].id, quantity: 1 }],
      }),
      { headers }
    );

    orderCreationDuration.add(Date.now() - orderStart);

    const orderSuccess = check(orderRes, {
      'order created': r => r.status === 201,
      'order has id': r => r.json('id') !== undefined,
    });

    if (!orderSuccess) {
      orderCreationErrors.add(1);
      console.error(`Order failed: ${orderRes.status} - ${orderRes.body}`);
    }

    successRate.add(orderSuccess);
  });

  sleep(1); // ユーザーの操作間隔をシミュレート
}

// サマリー出力のカスタマイズ
export function handleSummary(data) {
  return {
    'k6/results/summary.json': JSON.stringify(data, null, 2),
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
  };
}
```

---

## GitHub Actionsでの自動実行

```yaml
# .github/workflows/load-test.yml
name: Load Test

on:
  workflow_dispatch:
    inputs:
      base_url:
        description: 'Target URL'
        required: true
        default: 'https://staging.example.com'

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1

      - name: Run load test
        run: k6 run --vus 50 --duration 5m k6/order-load-test.js
        env:
          BASE_URL: ${{ github.event.inputs.base_url }}

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: k6/results/
```

---

## まとめ

Claude Codeで負荷テストを設計する：

1. **CLAUDE.md** にSLA目標（P95<500ms、エラー<0.1%）とテスト種別を明記
2. **stages** で段階的にVUを増加（急激な増加はスパイクテスト）
3. **thresholds** でSLA違反時にテストを自動失敗にする
4. **カスタムメトリクス** でビジネスロジックに関わるメトリクスを追跡

---

*APIのパフォーマンスボトルネックを検出するスキルは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
