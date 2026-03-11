---
title: "Claude CodeでREST APIを一貫した設計にする：規約の定義から自動チェックまで"
emoji: "🌐"
type: "tech"
topics: ["claudecode", "api", "rest", "typescript", "設計"]
published: true
---

## はじめに

REST APIの設計は、チームや時期によってバラバラになりやすい。`/user`と`/users`が混在したり、POSTで削除が行われたりする。

Claude CodeのCLAUDE.mdにREST規約を明示することで、常に一貫した設計のコードが生成されるようになる。

---

## CLAUDE.mdにREST設計規約を書く

```markdown
## REST API設計規約

### URL命名規則
- リソースは必ず複数形: `/users`、`/posts`（`/user`は禁止）
- ネストは最大2階層: `/users/{id}/posts`（それ以上は禁止）
- 動詞はRESTで表現できない操作のみ: `/orders/{id}/cancel`
- APIバージョンはURLに含める: `/api/v1/`

### HTTPメソッドの正しい使い方
- GET: データの読み取りのみ（副作用を持たせない）
- POST: 新規リソースの作成
- PUT: リソース全体の置換（全フィールド必須）
- PATCH: リソースの部分更新（変更フィールドのみ）
- DELETE: リソースの削除（ボディを含めない）

### レスポンス形式（統一必須）
- 成功（単体）: `{ data: T }`
- 成功（一覧）: `{ data: T[], pagination: { nextCursor, hasMore } }`
- エラー: `{ error: string, code?: string }`
- タイムスタンプ: ISO 8601形式（UTC）

### ステータスコード
- 200: 成功（GET/PUT/PATCH）
- 201: 作成成功（POST）
- 204: No Content（DELETE）
- 400: バリデーションエラー
- 401: 未認証
- 403: 認証済みだが権限なし
- 404: リソースなし
- 409: 競合（重複や楽観ロック失敗）
- 500: サーバーエラー

### ページネーション
- 大テーブル（>10万件）: カーソルベース
- 小テーブル: オフセット（page, limit）
- デフォルト: 20件、最大: 100件
```

---

## APIレビューを依頼する

設計した後にClaude Codeでレビュー：

```
以下のAPIエンドポイントをREST設計規約の観点でレビューしてください。
問題点があれば修正案も提示してください。

GET /api/v1/user — 全ユーザー取得
POST /api/v1/user/create — ユーザー作成
GET /api/v1/user/posts — ユーザーの投稿（IDなし）
DELETE /api/v1/user/remove/{id} — ユーザー削除
```

期待されるフィードバック：
- `/user` → `/users` （複数形）
- `POST /user/create` → `POST /users` （URLに動詞禁止）
- ユーザーIDが指定されていない
- `DELETE /user/remove/{id}` → `DELETE /users/{id}` （動詞禁止）

---

## CRUDエンドポイントの生成

```
PostリソースのCRUDエンドポイントをCLAUDE.mdの規約に従って生成してください。

Postのフィールド：
- id (UUID), title, content, authorId
- publishedAt (nullable), createdAt, updatedAt

必要なエンドポイント：
- 一覧（カーソルページネーション付き）
- 単体取得
- 作成・部分更新・削除
- 公開アクション（/posts/{id}/publish）
```

---

## Zodによる入力バリデーション

```
POST /users と PATCH /users/{id} のZodスキーマを生成してください。

POST要件：
- email: 有効なメール形式
- name: 1-100文字（前後のスペースを除去）
- role: 'admin' | 'user' | 'viewer' の列挙型

PATCH要件：
- 全フィールドをオプション（partial()を使う）
- 値が提供された場合は同じバリデーションを適用
```

---

## レスポンス型の生成

```
以下のAPIのレスポンス型をTypeScriptで生成してください。
CLAUDE.mdの形式（{ data: T }）に従うこと。
Prismaのモデルからは機密フィールド（passwordHash等）を除外すること。

Prismaモデル：
[User/Post等のモデルをここに貼る]
```

---

## OpenAPIドキュメントの生成

```
これらのExpressルートからOpenAPI 3.0のYAMLを生成してください。
ルートハンドラーのコードからリクエスト・レスポンスのスキーマを推測してください。

[ルーターファイルをここに貼る]
```

---

## まとめ

Claude CodeでREST APIを一貫して設計するために：

1. **CLAUDE.md** にURL命名・HTTPメソッド・レスポンス形式を定義
2. **設計後にレビューを依頼**（問題を早期発見）
3. **Zodスキーマを自動生成**（規約に準拠）
4. **OpenAPIを自動生成**（ドキュメントを常に最新に）

---

*REST規約への準拠を自動チェックするスキルは **Code Review Pack（¥980）** として PromptWorks で購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
