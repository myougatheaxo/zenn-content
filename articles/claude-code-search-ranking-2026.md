---
title: "Claude Codeで検索ランキングを設計する：BM25・パーソナライズ・クリックスルー学習"
emoji: "📊"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "elasticsearch", "redis"]
published: true
published_at: "2026-03-13 12:00"
---

## はじめに

「検索結果の1位が実は人気がない」——BM25テキスト関連度スコアにユーザー行動データ（クリック・購入）を組み合わせてパーソナライズされた検索ランキングを実現する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdに検索ランキング設計ルールを書く

```markdown
## 検索ランキング設計ルール

### スコアリング構成
- テキスト関連度 (60%): ElasticsearchのBM25
- 人気度 (25%): 過去30日のクリック数・購入数
- 新鮮度 (10%): 登録日からの減衰スコア
- パーソナライズ (5%): ユーザーの閲覧履歴ベース

### 学習ループ
- クリックイベントをRedisにリアルタイム蓄積
- 1時間ごとに集計してESのpopularityスコアを更新
- A/Bテストでランキングアルゴリズムを検証

### キャッシュ戦略
- 人気クエリ（上位100件）: Redisに10分キャッシュ
- ロングテールクエリ: キャッシュなし（毎回ES検索）
```

---

## 検索ランキングシステムの生成

```
商品検索ランキングシステムを設計してください。

要件：
- Elasticsearch BM25スコア
- 人気度スコア（クリック・購入）
- パーソナライズランキング
- クリックイベント収集
- A/Bテスト対応

生成ファイル: src/search/
```

---

## 生成される検索ランキング実装

```typescript
// src/search/rankingService.ts

interface SearchParams {
  query: string;
  userId?: string;     // パーソナライズ用（オプション）
  category?: string;
  page: number;
  size: number;
}

export class RankingService {
  async search(params: SearchParams): Promise<SearchResult> {
    const { query, userId, category, page, size } = params;

    // 人気クエリはRedisキャッシュを確認
    const cacheKey = `search:${query}:${category ?? 'all'}:${page}`;
    const popularQuery = await this.isPopularQuery(query);
    if (popularQuery) {
      const cached = await redis.get(cacheKey);
      if (cached) return JSON.parse(cached);
    }

    // ユーザーの閲覧履歴を取得（パーソナライズ用）
    const userPreferences = userId
      ? await this.getUserPreferences(userId)
      : null;

    // Elasticsearchのfunctionスコアクエリで複合ランキング
    const esQuery = this.buildRankingQuery(query, category, userPreferences);

    const result = await elasticsearch.search({
      index: 'products',
      body: esQuery,
      from: (page - 1) * size,
      size,
    });

    const searchResult = this.formatResult(result, params);

    // 人気クエリは10分キャッシュ
    if (popularQuery) {
      await redis.set(cacheKey, JSON.stringify(searchResult), { EX: 600 });
    }

    return searchResult;
  }

  private buildRankingQuery(
    query: string,
    category: string | undefined,
    userPrefs: UserPreferences | null
  ) {
    return {
      query: {
        function_score: {
          // メインクエリ: BM25テキスト関連度
          query: {
            bool: {
              must: [{
                multi_match: {
                  query,
                  fields: ['name^3', 'description^1', 'category^2'],
                  type: 'best_fields',
                  fuzziness: 'AUTO', // タイポ許容
                },
              }],
              filter: category ? [{ term: { category } }] : [],
            },
          },
          functions: [
            // 人気度スコア（クリック数・購入数ベース）
            {
              field_value_factor: {
                field: 'popularityScore',
                modifier: 'log1p', // 対数スケール（大きな差を均す）
                factor: 2.5,
                missing: 1,
              },
              weight: 0.25,
            },
            // 新鮮度スコア（登録から時間経過で減衰）
            {
              gauss: {
                createdAt: {
                  origin: 'now',
                  scale: '30d',    // 30日で50%
                  decay: 0.5,
                },
              },
              weight: 0.10,
            },
            // パーソナライズ（ユーザーの好みカテゴリを優先）
            ...(userPrefs?.topCategories.map(cat => ({
              filter: { term: { category: cat.name } },
              weight: 0.05 * cat.weight,
            })) ?? []),
          ],
          score_mode: 'sum',
          boost_mode: 'multiply', // 最終スコア = BM25スコア × functionスコア
        },
      },
      // ハイライト（検索キーワードを強調表示）
      highlight: {
        fields: {
          name: { pre_tags: ['<mark>'], post_tags: ['</mark>'] },
          description: { fragment_size: 150, number_of_fragments: 1 },
        },
      },
    };
  }
}
```

```typescript
// src/search/clickTracker.ts — クリックイベント収集と学習

// 検索クリックイベントをRedisに記録
export async function trackSearchClick(params: {
  query: string;
  productId: string;
  position: number;   // 検索結果の表示順位
  userId?: string;
}): Promise<void> {
  const now = Date.now();
  const hourKey = Math.floor(now / 3_600_000); // 1時間ごとのバケット

  const pipeline = redis.pipeline();

  // 商品のクリックカウント（1時間ごと）
  pipeline.zincrby(`clicks:products:${hourKey}`, 1, params.productId);
  pipeline.expire(`clicks:products:${hourKey}`, 86400 * 7); // 7日保持

  // クエリ別クリック（人気クエリ判定用）
  pipeline.zincrby('popular:queries', 1, params.query);

  // ユーザーの閲覧履歴（パーソナライズ用）
  if (params.userId) {
    pipeline.zadd(`user:history:${params.userId}`, now, params.productId);
    pipeline.zremrangebyrank(`user:history:${params.userId}`, 0, -101); // 最新100件のみ保持
    pipeline.expire(`user:history:${params.userId}`, 86400 * 30); // 30日保持
  }

  await pipeline.exec();
}

// 1時間ごとにESのpopularityスコアを更新
export async function updatePopularityScores(): Promise<void> {
  const hourKey = Math.floor(Date.now() / 3_600_000) - 1; // 前の1時間

  // 上位1000商品のクリック数を取得
  const clicks = await redis.zrangebyscore(
    `clicks:products:${hourKey}`,
    '-inf', '+inf',
    'WITHSCORES',
    'LIMIT', '0', '1000'
  );

  // ESのpopularityScoreをバルク更新
  const bulkOps = [];
  for (let i = 0; i < clicks.length; i += 2) {
    const productId = clicks[i];
    const clickCount = parseFloat(clicks[i + 1]);

    bulkOps.push({ update: { _index: 'products', _id: productId } });
    bulkOps.push({
      script: {
        source: 'ctx._source.popularityScore = ctx._source.popularityScore * 0.9 + params.clicks * 0.1',
        params: { clicks: clickCount },
      },
    });
  }

  if (bulkOps.length > 0) {
    await elasticsearch.bulk({ body: bulkOps });
    logger.info({ updated: bulkOps.length / 2 }, 'Popularity scores updated');
  }
}
```

---

## まとめ

Claude Codeで検索ランキングを設計する：

1. **CLAUDE.md** にBM25 60%・人気度 25%・新鮮度 10%・パーソナライズ 5%の配分を明記
2. **ESのfunction_score** でテキスト関連度×人気度×新鮮度の複合スコアリング
3. **Redis Sorted Set** でクリックイベントを1時間バケットに蓄積
4. **指数移動平均** でpopularityScoreを更新（古いデータを0.9倍でフェード）

---

*検索ランキング設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
