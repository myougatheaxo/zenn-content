---
title: "Claude Codeで全文検索を設計する：MeiliSearchとPostgreSQL全文検索"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "meilisearch", "postgresql"]
published: true
---

## はじめに

SQLの`LIKE '%keyword%'`は遅い——全テーブルスキャンになる。全文検索エンジンを使うと日本語も含めて高速に検索できる。Claude Codeに設計を生成させる。

---

## CLAUDE.mdに全文検索ルールを書く

```markdown
## 全文検索設計ルール

### エンジン選択
- 小規模(10万件以下): PostgreSQL全文検索（tsvector）で十分
- 中大規模・日本語重視: MeiliSearch（セルフホスト可）
- 大規模マネージド: Algolia or OpenSearch

### インデックス同期
- DBへの書き込みと同時に検索インデックスも更新
- 失敗時のリカバリー: バックグラウンドジョブで再同期
- 削除時は必ず検索インデックスからも削除

### 検索品質
- 日本語: MeiliSearch（形態素解析自動）
- タイポ許容: MeiliSearchのfuzzy searchを使う
- ハイライト: クライアントに検索結果のハイライトを返す
```

---

## MeiliSearch 全文検索の生成

```
MeiliSearchを使った全文検索を設計してください。

対象: 記事(Article)の検索
- title, content, tagsをインデックス
- DB書き込み時に同期
- 削除時にインデックスから削除
- タイポ許容、日本語対応

生成ファイル:
- src/services/searchService.ts
- src/routes/search.ts
```

---

## 生成されるMeiliSearch連携

```typescript
// src/services/searchService.ts
import { MeiliSearch } from 'meilisearch';

const meili = new MeiliSearch({
  host: process.env.MEILISEARCH_URL ?? 'http://localhost:7700',
  apiKey: process.env.MEILISEARCH_KEY,
});

const INDEX_NAME = 'articles';

// インデックス初期設定（起動時に1回実行）
export async function setupSearchIndex(): Promise<void> {
  const index = meili.index(INDEX_NAME);

  await index.updateSettings({
    searchableAttributes: ['title', 'content', 'tags'],
    filterableAttributes: ['authorId', 'publishedAt', 'tags'],
    sortableAttributes: ['publishedAt', 'viewCount'],
    rankingRules: [
      'words', 'typo', 'proximity', 'attribute', 'sort', 'exactness',
    ],
  });
}

export async function indexArticle(article: Article): Promise<void> {
  await meili.index(INDEX_NAME).addDocuments([
    {
      id: article.id,
      title: article.title,
      content: article.content.substring(0, 5000), // インデックスサイズ制限
      tags: article.tags,
      authorId: article.authorId,
      publishedAt: article.publishedAt?.getTime(),
    },
  ]);
}

export async function removeArticleFromIndex(articleId: string): Promise<void> {
  await meili.index(INDEX_NAME).deleteDocument(articleId);
}

export async function searchArticles(
  query: string,
  options: { limit?: number; offset?: number; tags?: string[] }
): Promise<SearchResult> {
  const filter = options.tags?.length ? `tags IN [${options.tags.map(t => `"${t}"`).join(', ')}]` : undefined;

  const result = await meili.index(INDEX_NAME).search(query, {
    limit: options.limit ?? 20,
    offset: options.offset ?? 0,
    filter,
    attributesToHighlight: ['title', 'content'],
    highlightPreTag: '<mark>',
    highlightPostTag: '</mark>',
  });

  return {
    hits: result.hits,
    total: result.estimatedTotalHits,
    processingTime: result.processingTimeMs,
  };
}
```

```typescript
// DB書き込みと同時にインデックスを更新
export async function createArticle(data: CreateArticleInput): Promise<Article> {
  const article = await prisma.article.create({ data });

  // 非同期でインデックス更新（失敗してもDBは成功扱い）
  indexArticle(article).catch(err =>
    logger.error({ err, articleId: article.id }, 'Failed to index article')
  );

  return article;
}
```

---

## PostgreSQL 全文検索（小規模向け）

```sql
-- tsvectorカラムとインデックスを追加
ALTER TABLE articles ADD COLUMN search_vector tsvector;

CREATE INDEX articles_search_idx ON articles USING GIN(search_vector);

-- トリガーで自動更新
CREATE FUNCTION update_search_vector() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := to_tsvector('japanese', COALESCE(NEW.title, '') || ' ' || COALESCE(NEW.content, ''));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_articles_search
  BEFORE INSERT OR UPDATE ON articles
  FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

```typescript
// Prismaで全文検索
const articles = await prisma.$queryRaw<Article[]>`
  SELECT * FROM articles
  WHERE search_vector @@ plainto_tsquery('japanese', ${query})
  ORDER BY ts_rank(search_vector, plainto_tsquery('japanese', ${query})) DESC
  LIMIT 20
`;
```

---

## まとめ

Claude Codeで全文検索を設計する：

1. **CLAUDE.md** にエンジン選択基準・同期戦略・検索品質要件を明記
2. **MeiliSearch** で日本語・タイポ許容・ハイライト付き検索
3. **DB書き込み時に同期** （失敗しても本体処理は続行）
4. **PostgreSQL tsvector** で小規模なら追加インフラ不要

---

*全文検索の設計レビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
