---
title: "第6章 テスト戦略 — バックエンドのテストをClaude Codeで自動化"
---

# 第6章 テスト戦略 — バックエンドのテストをClaude Codeで自動化

「テストを書く時間がない」はもう言い訳にならない。Claude Codeが実装と同時にテストも生成してくれる。

---

## 6-1. テストの三層構造

バックエンドのテストは3種類を組み合わせる:

```
ユニットテスト（多い）
  └─ 個別の関数・クラスを独立してテスト
     速い、安い、フィードバックが早い

統合テスト（中間）
  └─ DBやRedisを含めた複数コンポーネントの連携テスト
     実際の動作に近い

E2Eテスト（少ない）
  └─ APIエンドポイント全体のテスト
     最も信頼できるが遅い、高コスト
```

---

## 6-2. Claude Codeへのテスト生成依頼

実装と同時にテストを書かせる:

```bash
claude "src/services/product.service.ts の ProductService クラスの
ユニットテストをVitestで作成。

テストすべき動作:
1. getProduct: キャッシュヒット時はRedisから返す
2. getProduct: キャッシュミス時はDBから取得してキャッシュに保存
3. getProduct: Redisエラー時はDBにfallbackする
4. getProduct: DBにも存在しない場合はnullを返す
5. createProduct: バリデーションエラー時は AppError をthrowする

モック:
- PrismaClient → jest.mock or vitest mock
- Redis → インメモリ実装を使う"
```

---

## 6-3. APIテストの実装

Supertestを使ったAPIテスト:

```bash
claude "src/routes/users.ts のAPIルートをSupertestでテスト:

テストケース:
1. GET /api/users/:id - 正常系（ユーザーが存在する）
2. GET /api/users/:id - ユーザーが存在しない（404）
3. POST /api/users - バリデーションエラー（400、エラー詳細含む）
4. POST /api/users - メールアドレス重複（409）
5. PUT /api/users/:id - 認証なし（401）
6. PUT /api/users/:id - 他のユーザーのリソース（403）

テストDB: Docker + PostgreSQL（testcontainersを使う）"
```

実装例:

```typescript
// test/routes/users.test.ts
import request from 'supertest'
import { app } from '../../src/app'
import { prisma } from '../../src/db'

describe('GET /api/users/:id', () => {
  let testUser: User

  beforeEach(async () => {
    testUser = await prisma.user.create({
      data: { email: 'test@example.com', name: 'Test User' }
    })
  })

  afterEach(async () => {
    await prisma.user.deleteMany()
  })

  it('存在するユーザーを返す', async () => {
    const response = await request(app)
      .get(`/api/users/${testUser.id}`)
      .set('Authorization', `Bearer ${generateTestToken(testUser.id)}`)
      .expect(200)

    expect(response.body.id).toBe(testUser.id)
    expect(response.body.email).toBe(testUser.email)
  })

  it('存在しないユーザーは404', async () => {
    const response = await request(app)
      .get('/api/users/nonexistent-id')
      .set('Authorization', `Bearer ${generateTestToken('user-id')}`)
      .expect(404)

    expect(response.body.error.code).toBe('NOT_FOUND')
  })
})
```

---

## 6-4. テストのCIパイプライン

```bash
claude "GitHub ActionsでバックエンドのCI/CDを設定:

ステップ:
1. コードの checkout
2. 依存関係のインストール（npm ci）
3. PostgreSQL + Redis のサービスコンテナ起動
4. DBマイグレーション実行
5. ユニットテスト実行（vitest）
6. 統合テスト実行（supertest）
7. カバレッジレポート生成
8. カバレッジが80%未満の場合はfail

ブランチ戦略:
- feature/* → テストのみ
- main → テスト + デプロイ"
```

---

## 6-5. テストカバレッジの改善

```bash
claude "現在のテストカバレッジレポートを分析:
npm run test -- --coverage

カバレッジが低い（50%未満）のファイルを特定して、
優先度の高いテストケースを3つ提案して。

判断基準:
- ビジネスクリティカルな処理（決済、認証等）は優先度高
- エラーハンドリングのパスは見落としやすい
- Happy path だけでなく edge case も"
```

---

## まとめ

テスト自動化でClaude Codeが最も活躍するポイント:

1. **テストの骨格生成**: テストケースの一覧を伝えるだけで実装してくれる
2. **モックの設定**: 複雑なモックも「PrismaとRedisをモック」と指示するだけ
3. **カバレッジ改善**: 低カバレッジのファイルを分析して追加すべきテストを提案

次の章は、本書のまとめと今後の活用方法。
