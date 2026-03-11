---
title: "第1章 REST API設計 — Claude Codeと作る一貫したAPI"
---

# 第1章 REST API設計 — Claude Codeと作る一貫したAPI

バックエンドの土台はAPIだ。URLの設計、HTTPメソッドの使い方、エラーレスポンスの形式。これらが一貫していないと、フロントエンドとの連携でつまずく。

Claude Codeを使えば、設計レビューからドキュメント自動生成まで一気通貫でできる。

---

## 1-1. REST原則をClaude Codeに守らせる

まず基本的なREST設計の問題を検出させる:

```bash
claude "src/routes/ 以下のAPIエンドポイントをレビューして。
以下の問題があれば指摘:
1. URLに動詞が含まれている（/getUser, /createPost等）
2. 不適切なHTTPメソッドの使用（GETで副作用、POST for delete等）
3. ネストが2階層を超えている
4. ステータスコードの誤用（成功に400、エラーに200等）"
```

よくある問題パターン:

```typescript
// ❌ 問題のある設計
router.post('/api/getUser', handler)         // GET を使うべき
router.get('/api/deleteUser/:id', handler)   // DELETE を使うべき
router.post('/api/users/:id/activate', handler) // PATCH /users/:id/status

// ✅ 正しい設計
router.get('/api/users/:id', handler)
router.delete('/api/users/:id', handler)
router.patch('/api/users/:id/status', handler)
```

---

## 1-2. エラーレスポンスの統一

APIのエラーレスポンスが統一されていないと、フロントエンドで毎回違う処理が必要になる。

Claude Codeにミドルウェアとして実装させる:

```bash
claude "全エンドポイントで統一されたエラーレスポンスを返す
Expressミドルウェアを作成。

形式:
{
  error: {
    code: string,        // VALIDATION_ERROR, NOT_FOUND等
    message: string,     // ユーザー向けメッセージ
    details?: array,     // バリデーションエラーの詳細
    requestId: string    // トラブルシューティング用ID
  }
}

HTTP status codeと error.code のマッピングも定義"
```

実装例:

```typescript
// src/middleware/error-handler.ts
export class AppError extends Error {
  constructor(
    public code: string,
    public message: string,
    public statusCode: number,
    public details?: unknown[]
  ) {
    super(message)
  }
}

export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  _next: NextFunction
) => {
  const requestId = req.headers['x-request-id'] as string || generateId()

  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        details: err.details,
        requestId,
      }
    })
  }

  // 想定外のエラー
  console.error({ err, requestId })
  return res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'サーバーエラーが発生しました',
      requestId,
    }
  })
}
```

---

## 1-3. OpenAPIドキュメントの自動生成

コードを書いたらドキュメントも自動生成させる:

```bash
claude "src/routes/ 以下のExpressルートから
OpenAPI 3.0形式のYAMLを生成。

含めるもの:
- 全エンドポイント（path, method, summary）
- リクエストボディのスキーマ（TypeScriptの型定義から抽出）
- レスポンスのスキーマ（成功・エラー両方）
- 認証要件（Bearer token が必要なもの）
- クエリパラメータの定義

出力先: docs/openapi.yaml"
```

---

## 1-4. バージョニング戦略

APIバージョニングは早めに決める。後から変えるのが大変なので:

```bash
claude "現在のAPIにバージョニングを追加するための
リファクタリング計画を立てて。

方針:
- URLパスベース: /api/v1/users
- 後方互換性を維持しながら v2 を追加できる構造
- v1 のエンドポイントは最低6ヶ月は維持

src/routes/ の変更に加えて:
- ルーターの再構成
- バージョン別のミドルウェア設定
- deprecation警告の仕組み

を提案して"
```

---

## 1-5. ページネーションの一貫した実装

リスト系APIは必ずページネーションを実装する。2種類あるので選択:

| 方式 | 特徴 | 適したケース |
|------|------|------------|
| **Offset** | シンプル、ページ番号で指定 | 管理画面、ページ数が固定 |
| **Cursor** | 一貫性が高い、高速 | SNSフィード、リアルタイム更新 |

```bash
claude "src/controllers/user.ts のユーザー一覧APIに
cursor-based paginationを実装。

要件:
- cursor: 最後に取得したアイテムのID（base64エンコード）
- limit: 1回の取得件数（デフォルト20、最大100）
- hasNextPage: 次のページが存在するか
- レスポンス形式:
  { data: User[], pagination: { nextCursor: string|null, hasNextPage: boolean } }"
```

---

## まとめ

Claude Codeを使ったAPI設計の流れ:

1. **設計レビュー**: `claude "src/routes/ のAPIをRESTの観点でレビュー"`
2. **エラーハンドリング統一**: ミドルウェアとして1回実装
3. **OpenAPI生成**: ルートが完成したら自動生成
4. **バージョニング**: 初期から設計に組み込む

次の章では、このAPIに認証・認可を追加する。
