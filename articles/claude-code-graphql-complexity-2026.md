---
title: "Claude CodeでGraphQLクエリ複雑度制限を設計する：Depth制限・Cost分析・introspection保護"
emoji: "🔬"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture"]
published: true
published_at: "2026-03-21 14:00"
---

## はじめに

「悪意あるクライアントが深くネストしたGraphQLクエリでサーバーをダウンさせた」「N+1クエリが大量に発生してDBが詰まる」——クエリ深度制限・コスト分析・レート制限でGraphQL APIを保護する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにGraphQL保護設計ルールを書く

```markdown
## GraphQL保護設計ルール

### クエリ複雑度制限
- 深度制限: maxDepth=7（ネストが深すぎるクエリを拒否）
- コスト制限: maxCost=1000（フィールド数×コスト係数の合計）
- introspection: 本番は無効化（スキーマ情報を隠す）

### フィールドコスト設定
- スカラーフィールド: cost=1
- リストフィールド: cost=10 + 各要素のコスト
- 外部APIを呼ぶフィールド: cost=50
- 計算コストの高いフィールド: cost=100

### ペルシスティッドクエリ
- 本番では許可リスト（ホワイトリスト）のクエリのみ実行
- ランダムなクエリを拒否
- Automatic Persisted Queries(APQ)でクライアントを対応
```

---

## GraphQL複雑度制限実装の生成

```
GraphQLクエリ保護を設計してください。

要件：
- クエリ深度チェック
- コスト計算
- レート制限
- introspection保護

生成ファイル: src/graphql/plugins/
```

---

## 生成されるGraphQL複雑度制限実装

```typescript
// src/graphql/plugins/queryComplexity.ts — クエリ複雑度制限

import { ApolloServerPlugin, GraphQLRequestListener } from '@apollo/server';
import { parse, DocumentNode, FieldNode, SelectionSetNode } from 'graphql';

// フィールドのコスト定義
const FIELD_COSTS: Record<string, Record<string, number>> = {
  Query: {
    user: 1,
    users: 10,          // リスト: 高コスト
    searchUsers: 20,    // 検索: さらに高コスト
    orderHistory: 10,
    analytics: 100,     // 計算コスト高
  },
  User: {
    id: 1,
    email: 1,
    orders: 15,         // リスト: 高コスト
    paymentMethods: 10,
  },
  Order: {
    id: 1,
    items: 10,
    shippingAddress: 1,
    user: 5,            // 外部参照
  },
};

const DEFAULT_FIELD_COST = 1;
const LIST_MULTIPLIER = 10;  // リスト取得は10倍

export interface ComplexityConfig {
  maxDepth: number;         // 最大深度
  maxCost: number;          // 最大コスト
  defaultListSize: number;  // コスト計算でのデフォルトリストサイズ
}

function calculateComplexity(
  selectionSet: SelectionSetNode | undefined,
  typeName: string,
  config: ComplexityConfig,
  currentDepth = 0
): { depth: number; cost: number } {
  if (!selectionSet || currentDepth >= config.maxDepth) {
    return { depth: currentDepth, cost: 0 };
  }

  let totalCost = 0;
  let maxDepth = currentDepth;

  for (const selection of selectionSet.selections) {
    if (selection.kind !== 'Field') continue;

    const field = selection as FieldNode;
    const fieldName = field.name.value;
    const fieldCost = FIELD_COSTS[typeName]?.[fieldName] ?? DEFAULT_FIELD_COST;

    // リスト引数でのコスト調整
    const limitArg = field.arguments?.find(a => a.name.value === 'limit' || a.name.value === 'first');
    const limitValue = limitArg && limitArg.value.kind === 'IntValue'
      ? parseInt(limitArg.value.value)
      : config.defaultListSize;

    const isListField = fieldCost >= 10;  // コスト10以上はリストと判断
    const adjustedCost = isListField ? fieldCost * (limitValue / config.defaultListSize) : fieldCost;

    totalCost += adjustedCost;

    // ネストしたフィールドを再帰処理
    if (field.selectionSet) {
      // 次のタイプ名を推測（実際の実装では GraphQLSchema を参照）
      const nestedType = getNestedTypeName(typeName, fieldName);
      const nested = calculateComplexity(field.selectionSet, nestedType, config, currentDepth + 1);
      totalCost += nested.cost * (isListField ? limitValue : 1);
      maxDepth = Math.max(maxDepth, nested.depth);
    }
  }

  return { depth: maxDepth, cost: totalCost };
}

export function createComplexityPlugin(config: ComplexityConfig): ApolloServerPlugin {
  return {
    async requestDidStart(): Promise<GraphQLRequestListener<any>> {
      return {
        async didResolveOperation({ request, document }) {
          const { depth, cost } = calculateComplexity(
            document.definitions[0]?.kind === 'OperationDefinition'
              ? document.definitions[0].selectionSet
              : undefined,
            'Query',
            config
          );

          logger.debug({ depth, cost, maxDepth: config.maxDepth, maxCost: config.maxCost },
            'GraphQL query complexity');

          if (depth > config.maxDepth) {
            throw new GraphQLError(`Query depth ${depth} exceeds maximum ${config.maxDepth}`, {
              extensions: { code: 'QUERY_TOO_DEEP', depth, maxDepth: config.maxDepth },
            });
          }

          if (cost > config.maxCost) {
            throw new GraphQLError(`Query cost ${cost} exceeds maximum ${config.maxCost}`, {
              extensions: { code: 'QUERY_TOO_COMPLEX', cost, maxCost: config.maxCost },
            });
          }
        },
      };
    },
  };
}
```

```typescript
// src/graphql/plugins/persistedQueries.ts — 本番用許可リストクエリ

export class PersistedQueryStore {
  // 許可するクエリのハッシュ→クエリ文字列マップ
  private readonly allowedQueries = new Map<string, string>();

  register(queryString: string): string {
    const hash = createHash('sha256').update(queryString).digest('hex');
    this.allowedQueries.set(hash, queryString);
    return hash;
  }

  get(hash: string): string | undefined {
    return this.allowedQueries.get(hash);
  }

  isAllowed(hash: string): boolean {
    return this.allowedQueries.has(hash);
  }
}

// 本番環境での許可リストプラグイン
export function createPersistedQueryPlugin(
  store: PersistedQueryStore,
  isProd: boolean
): ApolloServerPlugin {
  return {
    async requestDidStart(): Promise<GraphQLRequestListener<any>> {
      return {
        async didReceiveRequest({ request }) {
          if (!isProd) return;  // 開発環境は制限なし

          const hash = request.http?.headers.get('x-query-hash');

          if (!hash) {
            throw new GraphQLError('Production only accepts persisted queries. Include X-Query-Hash header.', {
              extensions: { code: 'PERSISTED_QUERY_REQUIRED' },
            });
          }

          if (!store.isAllowed(hash)) {
            throw new GraphQLError('Unknown persisted query hash', {
              extensions: { code: 'PERSISTED_QUERY_NOT_FOUND' },
            });
          }
        },
      };
    },
  };
}

// ApolloServer設定
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    createComplexityPlugin({
      maxDepth: 7,
      maxCost: 1000,
      defaultListSize: 10,
    }),
    // introspection保護
    ...(process.env.NODE_ENV === 'production' ? [{
      async requestDidStart(): Promise<GraphQLRequestListener<any>> {
        return {
          async didReceiveRequest({ request }) {
            const query = request.query ?? '';
            if (query.includes('__schema') || query.includes('__type')) {
              throw new GraphQLError('Introspection disabled in production', {
                extensions: { code: 'INTROSPECTION_DISABLED' },
              });
            }
          },
        };
      },
    }] : []),
  ],
  introspection: process.env.NODE_ENV !== 'production',
});

// GraphQLレート制限（IPベース）
app.use('/graphql', createRateLimiter({
  windowMs: 60_000,
  max: 100,  // 1分間100リクエスト
  keyGenerator: (req) => req.ip,
  handler: (req, res) => res.status(429).json({
    errors: [{ message: 'Too many requests', extensions: { code: 'RATE_LIMIT_EXCEEDED' } }],
  }),
}));
```

---

## まとめ

Claude CodeでGraphQLクエリ複雑度制限を設計する：

1. **CLAUDE.md** にmaxDepth=7・maxCost=1000・本番はintrospection無効化・リストフィールドのコストは高め（10倍）・本番は許可リストのみを明記
2. **コスト計算** でフィールドの重みを考慮したリソース消費を制限——スカラーは1点、リスト取得は10点×要素数。`limit=100`でusersを取得したら`10×100=1000点`。コスト上限を超えたら429で拒否
3. **深度制限（maxDepth=7）** でN+1爆発を防止——`user { orders { items { product { category { parent { ... } } } } } }`のような無限ネストを深度7で打ち切る。ほとんどの正当なクエリは深度5以内に収まる
4. **本番環境のintrospection無効化** でスキーマ情報を隠す——`__schema`クエリを拒否してAPIの内部構造を攻撃者から隠す。開発環境はそのまま有効にしてGraphiQLを使えるようにする

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
