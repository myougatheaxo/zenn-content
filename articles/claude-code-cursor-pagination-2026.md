---
title: "Claude Codeでカーソルページネーションを設計する：大量データの高速ページング・ソート安定性・無限スクロール"
emoji: "📄"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 17:00"
---

## はじめに

「OFFSET/LIMITページネーションが1000ページ目あたりから遅い」「ページ表示中に新データが追加されて同じ投稿が2回出た」——カーソルベースページネーションで大量データを高速に、かつ安定してページングする設計をClaude Codeに生成させる。

---

## CLAUDE.mdにカーソルページネーション設計ルールを書く

```markdown
## カーソルページネーション設計ルール

### カーソルの設計
- カーソル = ソートキーの値をBase64エンコードしたもの
- 複合ソート（created_at DESC, id DESC）で安定したページングを保証
- カーソルは不透明（クライアントはparse不要、そのまま次のリクエストに使う）

### クエリの効率化
- WHERE created_at < cursor_created_at OR (created_at = cursor_created_at AND id < cursor_id)
- id列にインデックスがあれば高速（OFFSET/LIMITと違いフルスキャン不要）
- 最後のページかどうかはlimit+1件取得して判定

### API設計
- レスポンス: { data: [...], nextCursor: string | null, hasMore: boolean }
- nextCursorがnullで全データ取得完了
- prevCursorは持たない（戻るはfirstPageに戻す設計を推奨）
```

---

## カーソルページネーション実装の生成

```
カーソルベースページネーションを設計してください。

要件：
- 複合ソートキーによる安定ページング
- Base64カーソルエンコード
- 双方向ページング
- GraphQL/REST両対応

生成ファイル: src/pagination/
```

---

## 生成されるカーソルページネーション実装

```typescript
// src/pagination/cursorPagination.ts — カーソルページネーションエンジン

export interface PageInfo {
  hasNextPage: boolean;
  hasPreviousPage: boolean;
  startCursor: string | null;
  endCursor: string | null;
}

export interface Connection<T> {
  edges: Array<{ node: T; cursor: string }>;
  pageInfo: PageInfo;
  totalCount?: number;
}

export interface CursorFields {
  [key: string]: unknown;  // ソートキーのフィールドと値
}

export class CursorEncoder {
  static encode(fields: CursorFields): string {
    return Buffer.from(JSON.stringify(fields)).toString('base64url');
  }

  static decode(cursor: string): CursorFields {
    try {
      return JSON.parse(Buffer.from(cursor, 'base64url').toString('utf-8'));
    } catch {
      throw new InvalidCursorError('Invalid cursor format');
    }
  }
}

export interface PaginationArgs {
  first?: number;   // 前向き（N件取得）
  after?: string;   // 前向き（このカーソルより後）
  last?: number;    // 後向き（N件取得）
  before?: string;  // 後向き（このカーソルより前）
}

export class CursorPaginator<T extends Record<string, unknown>> {
  constructor(
    private readonly defaultLimit: number = 20,
    private readonly maxLimit: number = 100
  ) {}

  // PostgreSQL用のWHERE条件を生成
  buildWhereClause(
    cursor: CursorFields | null,
    sortFields: Array<{ field: string; direction: 'ASC' | 'DESC' }>
  ): string {
    if (!cursor) return '';

    // 複合ソートのカーソル条件
    // 例: (created_at, id) DESC の場合
    // WHERE (created_at, id) < (cursorCreatedAt, cursorId)
    const pairs = sortFields.map(({ field }) => field).join(', ');
    const values = sortFields
      .map(({ field }) => {
        const val = cursor[field];
        return typeof val === 'string' ? `'${val}'` : val;
      })
      .join(', ');

    const operator = sortFields[0].direction === 'DESC' ? '<' : '>';

    return `AND (${pairs}) ${operator} (${values})`;
  }

  buildConnection(
    items: T[],
    args: PaginationArgs,
    cursorFields: Array<keyof T>
  ): Connection<T> {
    const limit = Math.min(args.first ?? args.last ?? this.defaultLimit, this.maxLimit);
    const hasMore = items.length > limit;

    // limit+1件取得して最後の1件を切り捨てる
    const edges = items.slice(0, limit).map(item => ({
      node: item,
      cursor: CursorEncoder.encode(
        Object.fromEntries(cursorFields.map(f => [f, item[f]]))
      ),
    }));

    return {
      edges,
      pageInfo: {
        hasNextPage: args.first !== undefined ? hasMore : false,
        hasPreviousPage: args.last !== undefined ? hasMore : !!args.after,
        startCursor: edges[0]?.cursor ?? null,
        endCursor: edges[edges.length - 1]?.cursor ?? null,
      },
    };
  }
}
```

```typescript
// src/pagination/postRepository.ts — 投稿一覧のカーソルページネーション

interface Post {
  id: string;
  title: string;
  createdAt: Date;
  authorId: string;
}

export class PostRepository {
  private readonly paginator = new CursorPaginator<Post>(20, 100);

  async findMany(args: PaginationArgs & { userId?: string }): Promise<Connection<Post>> {
    const limit = Math.min(args.first ?? args.last ?? 20, 100);

    let cursorWhere = '';
    if (args.after) {
      const cursor = CursorEncoder.decode(args.after);
      // (created_at, id) の複合カーソル（作成日時降順、同時刻はID降順）
      cursorWhere = `
        AND (p.created_at < '${cursor.createdAt}'
          OR (p.created_at = '${cursor.createdAt}' AND p.id < '${cursor.id}'))
      `;
    }

    // limit+1件取得（次ページ存在チェック用）
    const posts = await prisma.$queryRaw<Post[]>`
      SELECT p.id, p.title, p.created_at as "createdAt", p.author_id as "authorId"
      FROM posts p
      WHERE 1=1
        ${args.userId ? prisma.sql`AND p.author_id = ${args.userId}` : prisma.sql``}
        ${prisma.sql.raw(cursorWhere)}
      ORDER BY p.created_at DESC, p.id DESC
      LIMIT ${limit + 1}
    `;

    return this.paginator.buildConnection(posts, args, ['createdAt', 'id']);
  }
}

// REST APIエンドポイント
router.get('/api/posts', async (req, res) => {
  const repo = new PostRepository();

  const result = await repo.findMany({
    first: req.query.first ? parseInt(req.query.first as string) : 20,
    after: req.query.after as string | undefined,
  });

  // REST形式のレスポンス
  res.json({
    data: result.edges.map(e => e.node),
    pagination: {
      nextCursor: result.pageInfo.hasNextPage ? result.pageInfo.endCursor : null,
      hasMore: result.pageInfo.hasNextPage,
    },
  });
});

// GraphQLリゾルバー（GraphQL Connection仕様準拠）
const postsResolver = async (_: unknown, args: PaginationArgs) => {
  const repo = new PostRepository();
  return repo.findMany(args);
};

// 無限スクロールの使用例（フロントエンド側のロジック）
/*
let cursor: string | null = null;

async function loadMore() {
  const { data, pagination } = await fetch(`/api/posts?${cursor ? `after=${cursor}` : ''}`).then(r => r.json());

  appendPosts(data);
  cursor = pagination.nextCursor;

  if (!pagination.hasMore) {
    hideLoadMoreButton();
  }
}
*/
```

---

## まとめ

Claude Codeでカーソルページネーションを設計する：

1. **CLAUDE.md** にカーソル = Base64エンコードされたソートキー・複合ソート（created_at+id）で安定ページング・limit+1件取得で次ページ存在判定を明記
2. **複合カーソル条件** `(created_at, id) < (cursorCreatedAt, cursorId)` でOFFSET/LIMITのフルスキャンを回避——インデックスを使ってO(1)でカーソル位置にジャンプ。100万件のテーブルでも1ページ目と最後のページで速度差なし
3. **不透明なカーソル（Base64）** でクライアントがカーソルの内部構造に依存しない——ソートキーをcreated_at+idからscoreに変えてもAPIの互換性を維持できる
4. **limit+1件取得** でSELECT COUNTを使わずに`hasNextPage`を判定——COUNTは大テーブルで遅いため、limit+1件取得して「limit超えたかどうか」だけ見る

---

*DB設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
