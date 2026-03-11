---
title: "Claude CodeでGraphQL Persisted Queriesを設計する：APQ・クエリ許可リスト・帯域削減"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "graphql", "redis"]
published: true
published_at: "2026-03-16 19:00"
---

## はじめに

「GraphQLクエリが毎回全文送信されていてMobileの通信量が多い」「任意クエリを受け付けているのでセキュリティが心配」——Automatic Persisted Queries（APQ）とクエリ許可リストでGraphQLを最適化する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにPersisted Queries設計ルールを書く

```markdown
## GraphQL Persisted Queries設計ルール

### APQ（Automatic Persisted Queries）
- クライアントはクエリハッシュ（SHA256）のみを送信
- サーバーにハッシュが未登録なら4xx → クライアントが全クエリを再送
- Redisにhash→query文字列をキャッシュ（TTL: 7日）

### セキュリティ（本番環境）
- 許可リストモード: 登録済みクエリのみ実行可（任意クエリ禁止）
- 未登録クエリは403
- CI/CDでクエリをBuildTime登録（デプロイ時にサーバーへアップロード）

### 帯域最適化
- Mobile: GET?extensions={"persistedQuery":{"version":1,"sha256Hash":"..."}}
- Desktop: POST（APQフォールバック対応）
- クエリ圧縮: gzip（本番必須）
```

---

## Persisted Queries実装の生成

```
GraphQL Persisted Queriesシステムを設計してください。

要件：
- APQプロトコル実装
- Redisへのクエリキャッシュ
- 許可リストモード（本番環境）
- CI/CDでのクエリ登録スクリプト

生成ファイル: src/graphql/persistedQueries/
```

---

## 生成されるPersisted Queries実装

```typescript
// src/graphql/persistedQueries/apqPlugin.ts — APQプラグイン（Apollo Server）

import crypto from 'crypto';
import { ApolloServerPlugin, GraphQLRequestContext } from '@apollo/server';

export interface APQOptions {
  allowArbitraryQueries: boolean; // false = 本番許可リストモード
  ttlSeconds: number;             // Redisキャッシュ有効期限
}

export function apqPlugin(options: APQOptions): ApolloServerPlugin {
  return {
    async requestDidStart(): Promise<any> {
      return {
        async didResolveSource(requestContext: GraphQLRequestContext<any>) {
          const { request } = requestContext;
          const extensions = request.extensions as any;
          const persistedQuery = extensions?.persistedQuery;

          if (!persistedQuery) return; // 通常のクエリ

          const { sha256Hash, version } = persistedQuery;
          if (version !== 1) throw new Error('Unsupported APQ version');

          if (request.query) {
            // クライアントがハッシュ + クエリ全文を送ってきた場合: 登録
            const expectedHash = crypto.createHash('sha256').update(request.query).digest('hex');

            if (expectedHash !== sha256Hash) {
              throw new Error('Hash mismatch: client sent invalid APQ');
            }

            await redis.set(
              `apq:${sha256Hash}`,
              request.query,
              { EX: options.ttlSeconds }
            );

            logger.debug({ hash: sha256Hash }, 'APQ registered');
          } else {
            // クライアントがハッシュのみを送ってきた場合: 解決
            const cachedQuery = await redis.get(`apq:${sha256Hash}`);

            if (!cachedQuery) {
              // PersistedQueryNotFound: クライアントに全クエリを要求
              throw new PersistedQueryNotFoundError();
            }

            request.query = cachedQuery;
            logger.debug({ hash: sha256Hash }, 'APQ resolved from cache');
          }
        },
      };
    },
  };
}

class PersistedQueryNotFoundError extends Error {
  readonly extensions = { code: 'PERSISTED_QUERY_NOT_FOUND' };
  constructor() {
    super('PersistedQueryNotFound');
  }
}
```

```typescript
// src/graphql/persistedQueries/allowlistPlugin.ts — 許可リストモード（本番環境）

export interface AllowlistedQuery {
  hash: string;
  queryName: string;
  registeredAt: Date;
  clientName: string;
}

export class QueryAllowlist {
  private readonly allowlistKey = 'apq:allowlist';

  // CI/CDでクエリを登録（ビルド時）
  async register(queries: AllowlistedQuery[]): Promise<void> {
    const pipeline = redis.pipeline();
    for (const q of queries) {
      pipeline.hset(this.allowlistKey, q.hash, JSON.stringify(q));
    }
    await pipeline.exec();
    logger.info({ count: queries.length }, 'Queries registered to allowlist');
  }

  async isAllowed(hash: string): Promise<boolean> {
    return (await redis.hexists(this.allowlistKey, hash)) === 1;
  }

  async getAll(): Promise<AllowlistedQuery[]> {
    const data = await redis.hgetall(this.allowlistKey);
    return Object.values(data ?? {}).map(v => JSON.parse(v));
  }
}

// 許可リストモードのApolloプラグイン
export function allowlistPlugin(allowlist: QueryAllowlist, enforceInProd = true): ApolloServerPlugin {
  return {
    async requestDidStart() {
      return {
        async didResolveSource(requestContext) {
          const { request } = requestContext;
          const extensions = request.extensions as any;
          const hash = extensions?.persistedQuery?.sha256Hash;

          // ハッシュなし = 任意クエリ
          if (!hash) {
            if (enforceInProd && process.env.NODE_ENV === 'production') {
              throw new Error('Arbitrary queries are not allowed in production. Use persisted queries.');
            }
            return;
          }

          // 許可リストチェック
          const allowed = await allowlist.isAllowed(hash);
          if (!allowed) {
            logger.warn({ hash }, 'Unauthorized query hash');
            throw new Error(`Query hash ${hash} is not in the allowlist`);
          }
        },
      };
    },
  };
}
```

```typescript
// scripts/registerPersistedQueries.ts — CI/CDでの登録スクリプト

import { createHash } from 'crypto';
import { glob } from 'glob';
import { readFileSync } from 'fs';
import { parse, print } from 'graphql';

async function extractQueriesFromFiles(pattern: string): Promise<AllowlistedQuery[]> {
  const files = await glob(pattern);
  const queries: AllowlistedQuery[] = [];

  for (const file of files) {
    const content = readFileSync(file, 'utf-8');

    // ファイルからgql`...`を抽出
    const gqlMatches = content.matchAll(/gql`([\s\S]*?)`/g);

    for (const match of gqlMatches) {
      const queryStr = match[1];
      try {
        // 正規化（コメント除去・空白統一）
        const normalized = print(parse(queryStr));
        const hash = createHash('sha256').update(normalized).digest('hex');

        // 操作名を取得
        const nameMatch = queryStr.match(/(?:query|mutation|subscription)\s+(\w+)/);
        const queryName = nameMatch?.[1] ?? 'anonymous';

        queries.push({
          hash,
          queryName,
          registeredAt: new Date(),
          clientName: 'web-app',
        });
      } catch {
        // GraphQL以外のgqlタグはスキップ
      }
    }
  }

  return queries;
}

// CI/CDパイプラインからの実行
async function main() {
  const queries = await extractQueriesFromFiles('src/**/*.{ts,tsx}');

  console.log(`Found ${queries.length} GraphQL queries`);

  const allowlist = new QueryAllowlist();
  await allowlist.register(queries);

  console.log('Queries registered to allowlist');
  process.exit(0);
}

main().catch(err => { console.error(err); process.exit(1); });

// Apollo Server設定
const server = new ApolloServer({
  schema,
  plugins: [
    apqPlugin({ allowArbitraryQueries: false, ttlSeconds: 7 * 86400 }),
    allowlistPlugin(new QueryAllowlist(), process.env.NODE_ENV === 'production'),
  ],
});
```

---

## まとめ

Claude CodeでGraphQL Persisted Queriesを設計する：

1. **CLAUDE.md** にAPQプロトコル（ハッシュ→クエリ解決）・Redisに7日TTLキャッシュ・本番は許可リストモード・CI/CDでビルド時登録を明記
2. **PersistedQueryNotFoundError** でクライアントにフォールバックを促す——初回のみ全クエリ送信、以降はハッシュのみで通信量を大幅削減
3. **クエリ正規化（printとparse）** でコメントや空白の違いを吸収——同じクエリが異なるハッシュになることを防止
4. **CI/CDでbuild時にクエリ登録** ——ソースコードの変更なしにクエリが追加された場合はデプロイが失敗する防線として機能

---

*セキュリティ設計のレビューは **Security Pack（¥1,480）** の `/security-check` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
