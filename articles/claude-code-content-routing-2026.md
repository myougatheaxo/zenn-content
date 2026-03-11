---
title: "Claude Codeでコンテンツベースルーティングを設計する：ペイロード内容によるメッセージ振り分け・動的ルーティング"
emoji: "🔀"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-18 16:00"
---

## はじめに

「メッセージのペイロード内容によって送り先を変えたい」「高額注文は優先Workerへ、通常注文は通常Workerへ」——コンテンツベースルーティングで、メッセージの内容を評価して動的に送り先を決定する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにコンテンツベースルーティング設計ルールを書く

```markdown
## コンテンツベースルーティング設計ルール

### ルーティングルール
- ルールはコードではなく設定として外部化（DB/Redis）
- ルール評価は高速（JSONPathまたはシンプルな条件式）
- デフォルトルート: どのルールにも一致しない場合の送り先

### ルーティング先の種類
- キュー名（Redis Stream/RabbitMQ Exchange）
- サービスエンドポイント（HTTP POST）
- 複数送り先（Fanout的な複数ルート）

### 動的ルール更新
- ルールのホットリロード（再起動不要）
- ルール変更はRedisキャッシュを即座に無効化
- ルール評価のテスト機能（test modeでメッセージを渡してルートを確認）
```

---

## コンテンツベースルーティング実装の生成

```
コンテンツベースルーティングシステムを設計してください。

要件：
- 設定ベースのルーティングルール
- JSONPathによるペイロード評価
- 動的ルール更新
- ルーティングログ

生成ファイル: src/messaging/routing/
```

---

## 生成されるコンテンツベースルーティング実装

```typescript
// src/messaging/routing/contentRouter.ts — コンテンツベースルーター

export type RouteCondition =
  | { type: 'field_equals'; field: string; value: unknown }
  | { type: 'field_greater_than'; field: string; value: number }
  | { type: 'field_in'; field: string; values: unknown[] }
  | { type: 'field_exists'; field: string }
  | { type: 'regex'; field: string; pattern: string }
  | { type: 'and'; conditions: RouteCondition[] }
  | { type: 'or'; conditions: RouteCondition[] };

export interface RoutingRule {
  id: string;
  name: string;
  priority: number;         // 低いほど優先（0が最高優先）
  condition: RouteCondition;
  destination: string;      // ルーティング先（キュー名またはURL）
  destinationType: 'queue' | 'http' | 'multi';
  destinations?: string[];  // destinationType='multi'の場合
  enabled: boolean;
}

export class ContentRouter {
  private rulesCache: RoutingRule[] | null = null;
  private cacheExpiresAt = 0;
  private readonly CACHE_TTL_MS = 30_000;

  async route(message: { messageId: string; eventType: string; payload: unknown }): Promise<{
    destination: string | string[];
    ruleName: string | 'default';
  }> {
    const rules = await this.getRules();

    for (const rule of rules) {
      if (!rule.enabled) continue;

      if (this.evaluate(message.payload, rule.condition)) {
        logger.debug({ messageId: message.messageId, ruleName: rule.name, destination: rule.destination }, 'Message routed');
        metrics.routingDecision.inc({ rule: rule.name });

        const destination = rule.destinationType === 'multi'
          ? (rule.destinations ?? [rule.destination])
          : rule.destination;

        return { destination, ruleName: rule.name };
      }
    }

    // デフォルトルート
    return { destination: 'queue:default', ruleName: 'default' };
  }

  private evaluate(payload: unknown, condition: RouteCondition): boolean {
    const obj = payload as Record<string, unknown>;

    switch (condition.type) {
      case 'field_equals':
        return this.getField(obj, condition.field) === condition.value;

      case 'field_greater_than':
        return Number(this.getField(obj, condition.field)) > condition.value;

      case 'field_in':
        return condition.values.includes(this.getField(obj, condition.field));

      case 'field_exists':
        return this.getField(obj, condition.field) !== undefined;

      case 'regex': {
        const value = String(this.getField(obj, condition.field) ?? '');
        return new RegExp(condition.pattern).test(value);
      }

      case 'and':
        return condition.conditions.every(c => this.evaluate(payload, c));

      case 'or':
        return condition.conditions.some(c => this.evaluate(payload, c));

      default:
        return false;
    }
  }

  // ネストされたフィールドへのアクセス（例: 'order.totalAmount'）
  private getField(obj: Record<string, unknown>, field: string): unknown {
    return field.split('.').reduce((current: unknown, key) => {
      if (current && typeof current === 'object') {
        return (current as Record<string, unknown>)[key];
      }
      return undefined;
    }, obj);
  }

  private async getRules(): Promise<RoutingRule[]> {
    if (this.rulesCache && Date.now() < this.cacheExpiresAt) {
      return this.rulesCache;
    }

    const rules = await prisma.routingRule.findMany({
      where: { enabled: true },
      orderBy: { priority: 'asc' },
    });

    this.rulesCache = rules.map(r => ({
      ...r,
      condition: JSON.parse(r.conditionJson),
    }));
    this.cacheExpiresAt = Date.now() + this.CACHE_TTL_MS;

    return this.rulesCache!;
  }

  // ルール変更時にキャッシュを無効化
  invalidateCache(): void {
    this.rulesCache = null;
    this.cacheExpiresAt = 0;
  }
}
```

```typescript
// src/messaging/routing/routingRuleExamples.ts — ルール定義例

// ルール設定例（DBに保存）
const ROUTING_RULES: RoutingRule[] = [
  {
    id: 'vip-high-value',
    name: 'VIPユーザーの高額注文を優先キューへ',
    priority: 0,
    condition: {
      type: 'and',
      conditions: [
        { type: 'field_equals', field: 'user.tier', value: 'VIP' },
        { type: 'field_greater_than', field: 'order.totalAmount', value: 100_000 },
      ],
    },
    destination: 'queue:orders-priority',
    destinationType: 'queue',
    enabled: true,
  },
  {
    id: 'international-orders',
    name: '海外注文を国際配送チームへ',
    priority: 1,
    condition: {
      type: 'field_in',
      field: 'shipping.country',
      values: ['US', 'GB', 'AU', 'CA', 'DE', 'FR'],
    },
    destination: 'queue:orders-international',
    destinationType: 'queue',
    enabled: true,
  },
  {
    id: 'fraud-risk-review',
    name: '詐欺リスク高の注文を審査キューと通知へ',
    priority: 2,
    condition: {
      type: 'field_greater_than',
      field: 'fraudScore',
      value: 0.7,
    },
    destinations: ['queue:orders-review', 'queue:fraud-alerts'],
    destination: 'queue:orders-review',
    destinationType: 'multi',
    enabled: true,
  },
];

// 管理者API
router.get('/api/admin/routing/rules', requireAdmin, async (req, res) => {
  const rules = await prisma.routingRule.findMany({ orderBy: { priority: 'asc' } });
  res.json(rules);
});

router.put('/api/admin/routing/rules/:id', requireAdmin, async (req, res) => {
  await prisma.routingRule.update({ where: { id: req.params.id }, data: req.body });

  // ルーター全インスタンスのキャッシュを無効化
  await redis.publish('routing:cache-invalidate', 'rules-updated');
  res.json({ message: 'Rule updated' });
});

// テストモード: メッセージをルーターに通してどのルールが一致するか確認
router.post('/api/admin/routing/test', requireAdmin, async (req, res) => {
  const router = new ContentRouter();
  const result = await router.route({ messageId: 'test', eventType: req.body.eventType, payload: req.body.payload });
  res.json({ matched: result });
});
```

---

## まとめ

Claude Codeでコンテンツベースルーティングを設計する：

1. **CLAUDE.md** にルールはDBに外部化してコード変更不要・JSONPath的な条件式で評価・デフォルトルートを必ず設定・ルール変更はキャッシュを即座に無効化を明記
2. **条件式のAND/OR合成** でネストした条件を宣言的に記述——「VIPユーザー かつ 10万円以上」を`{ type: 'and', conditions: [...] }`で表現し、コードを変えずにルールを変更可能
3. **multi destinationType** で同じメッセージを複数キューに同時ルーティング——詐欺リスクが高い注文を審査キュー+アラートキューの両方に送る（Fan-outとの組み合わせ）
4. **テストモードAPI** でルール変更前に動作確認——「このペイロードはどのルールに一致するか」を本番ルールで事前検証して意図しないルーティングを防止

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
