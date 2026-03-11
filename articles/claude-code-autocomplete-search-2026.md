---
title: "Claude Codeでオートコンプリート検索を設計する：サジェスト・タイプアヘッド・検索インデックス"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "elasticsearch"]
published: true
---

## はじめに

検索ボックスに入力するたびにリアルタイムでサジェストを表示する——デバウンス・トライ木・Redisキャッシュを組み合わせてバックエンドを最適化する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにオートコンプリート設計ルールを書く

```markdown
## オートコンプリート設計ルール

### フロントエンド
- デバウンス: 300ms（キー入力ごとにAPIを叩かない）
- 最小文字数: 2文字以上でサジェスト開始
- キャンセル: 古いリクエストはAbortControllerでキャンセル

### バックエンド
- Redis Sorted Set（ZSet）でプレフィックスインデックス管理
- Elasticsearchのcompletion suggesterを本番で使用
- サジェスト結果はRedisでキャッシュ（TTL: 5分）
- 返す件数: 最大10件

### インデックス戦略
- 商品追加時にリアルタイムインデックス更新
- 夜間バッチでインデックス最適化
- 人気順（スコア）でソート
```

---

## オートコンプリートシステムの生成

```
商品検索のオートコンプリートシステムを設計してください。

要件：
- Redis ZSetでプレフィックスインデックス
- タイプアヘッドサジェスト（最大10件）
- 人気度スコアによるランキング
- Elasticsearchとの統合

生成ファイル: src/search/autocomplete.ts
```

---

## 生成されるオートコンプリート実装

```typescript
// src/search/autocomplete.ts

// Redis ZSet でプレフィックスインデックスを管理
// キー: "autocomplete:products" / スコア: 人気度 / メンバー: "商品名::商品ID"

const AUTOCOMPLETE_KEY = 'autocomplete:products';
const CACHE_TTL = 300; // 5分

export class AutocompleteService {
  // 商品をオートコンプリートインデックスに追加
  async indexProduct(product: {
    id: string;
    name: string;
    category: string;
    popularityScore: number; // 0-1000（クリック数ベース）
  }): Promise<void> {
    const pipeline = redis.pipeline();

    // 商品名の全プレフィックスをインデックス（最低2文字から）
    const normalizedName = product.name.toLowerCase();
    for (let i = 2; i <= normalizedName.length; i++) {
      const prefix = normalizedName.slice(0, i);
      const member = `${normalizedName}::${product.id}`;

      // ZSet: スコア=人気度, メンバー=名前::ID
      pipeline.zadd(AUTOCOMPLETE_KEY, {
        score: product.popularityScore,
        member,
      });

      // プレフィックスごとのセット
      pipeline.zadd(`autocomplete:prefix:${prefix}`, {
        score: product.popularityScore,
        member,
      });
      pipeline.expire(`autocomplete:prefix:${prefix}`, 86400); // 1日
    }

    await pipeline.exec();
  }

  // プレフィックスからサジェストを取得
  async getSuggestions(
    prefix: string,
    options: { limit?: number; category?: string } = {}
  ): Promise<SuggestionResult[]> {
    const { limit = 10, category } = options;
    const normalizedPrefix = prefix.toLowerCase().trim();

    if (normalizedPrefix.length < 2) return [];

    // キャッシュチェック
    const cacheKey = `suggestions:${normalizedPrefix}:${category ?? 'all'}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // プレフィックスのインデックスから取得（スコア降順）
    const members = await redis.zrangebyscore(
      `autocomplete:prefix:${normalizedPrefix}`,
      '-inf',
      '+inf',
      'WITHSCORES',
      'LIMIT',
      '0',
      String(limit * 2) // フィルタリングのため多めに取得
    );

    // 結果を解析してフォーマット
    const suggestions: SuggestionResult[] = [];

    for (let i = 0; i < members.length; i += 2) {
      const member = members[i];
      const score = parseFloat(members[i + 1]);
      const [name, productId] = member.split('::');

      // DBから追加情報を取得（サムネイル等）
      const product = await this.getProductFromCache(productId);
      if (!product) continue;

      // カテゴリフィルタリング
      if (category && product.category !== category) continue;

      suggestions.push({
        id: productId,
        name: product.name,
        category: product.category,
        imageUrl: product.thumbnailUrl,
        price: product.priceCents,
        score,
        highlight: this.highlightMatch(product.name, normalizedPrefix),
      });

      if (suggestions.length >= limit) break;
    }

    // 結果をキャッシュ
    await redis.set(cacheKey, JSON.stringify(suggestions), { EX: CACHE_TTL });

    return suggestions;
  }

  // マッチ部分をハイライト（フロントエンド表示用）
  private highlightMatch(name: string, prefix: string): string {
    const lowerName = name.toLowerCase();
    const start = lowerName.indexOf(prefix);
    if (start === -1) return name;

    return (
      name.slice(0, start) +
      `<mark>${name.slice(start, start + prefix.length)}</mark>` +
      name.slice(start + prefix.length)
    );
  }

  // Elasticsearchのcompletion suggester（本番推奨）
  async getSuggestionsES(prefix: string, limit = 10): Promise<SuggestionResult[]> {
    const result = await elasticsearch.search({
      index: 'products',
      body: {
        suggest: {
          product_suggest: {
            prefix,
            completion: {
              field: 'name_suggest',  // ES mappingで定義したcompletion field
              size: limit,
              fuzzy: {
                fuzziness: 1,  // タイポを1文字まで許容
              },
              contexts: {
                // カテゴリによるフィルタリング
                status: ['active'],
              },
            },
          },
        },
      },
    });

    return result.suggest.product_suggest[0].options.map(opt => ({
      id: opt._id,
      name: opt._source.name,
      category: opt._source.category,
      imageUrl: opt._source.thumbnailUrl,
      price: opt._source.priceCents,
      score: opt._score,
      highlight: opt.highlighted ?? opt._source.name,
    }));
  }
}

export const autocomplete = new AutocompleteService();
```

---

## APIエンドポイント

```typescript
// src/routes/search.ts

// オートコンプリートAPI（デバウンスはフロントエンド側で実装）
router.get('/search/suggest', async (req, res) => {
  const query = req.query.q as string;
  const category = req.query.category as string | undefined;

  if (!query || query.length < 2) {
    return res.json({ suggestions: [] });
  }

  const suggestions = await autocomplete.getSuggestions(query, {
    limit: 10,
    category,
  });

  // CDNキャッシュ: 同じクエリは5分キャッシュ
  res.setHeader('Cache-Control', 'public, max-age=300, s-maxage=300');
  res.json({ suggestions });
});
```

---

## フロントエンドのデバウンス実装（React）

```typescript
// src/components/SearchBox.tsx
import { useState, useEffect, useRef } from 'react';

export function SearchBox() {
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState<SuggestionResult[]>([]);
  const abortRef = useRef<AbortController | null>(null);

  useEffect(() => {
    if (query.length < 2) {
      setSuggestions([]);
      return;
    }

    const timer = setTimeout(async () => {
      // 前のリクエストをキャンセル
      abortRef.current?.abort();
      abortRef.current = new AbortController();

      try {
        const res = await fetch(
          `/api/search/suggest?q=${encodeURIComponent(query)}`,
          { signal: abortRef.current.signal }
        );
        const data = await res.json();
        setSuggestions(data.suggestions);
      } catch (err) {
        if ((err as Error).name !== 'AbortError') {
          console.error(err);
        }
      }
    }, 300); // 300msデバウンス

    return () => clearTimeout(timer);
  }, [query]);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="商品を検索..."
      />
      {suggestions.length > 0 && (
        <ul>
          {suggestions.map(s => (
            <li key={s.id}>
              <span dangerouslySetInnerHTML={{ __html: s.highlight }} />
              <span>{s.category}</span>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## まとめ

Claude Codeでオートコンプリートを設計する：

1. **CLAUDE.md** にデバウンス300ms・最小2文字・Redis/ESインデックス管理を明記
2. **Redis ZSet** でプレフィックスインデックス（スコアで人気順ランキング）
3. **Elasticsearchのcompletion suggester** で本番スケールのサジェスト
4. **AbortController** で古いリクエストをキャンセル（レースコンディション防止）

---

*検索システム設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
