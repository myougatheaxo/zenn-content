---
title: "第4章 キャッシュ設計 — Redisで高速化しつつ一貫性を保つ"
---

# 第4章 キャッシュ設計 — Redisで高速化しつつ一貫性を保つ

キャッシュはパフォーマンス改善の切り札だが、一貫性の問題を引き起こしやすい。Claude Codeに「何をキャッシュすべきか」から実装まで任せる。

---

## 4-1. キャッシュ対象の選定

```bash
claude "以下のエンドポイントのキャッシュ戦略を提案:

1. GET /products - 商品一覧（全ユーザー共通）
2. GET /products/:id - 商品詳細
3. GET /users/:id/recommendations - ユーザー別おすすめ
4. GET /dashboard/stats - 管理画面の統計

各エンドポイントについて:
- キャッシュすべきか（更新頻度、パーソナライズ度で判断）
- TTL（有効期限）
- キャッシュキーの設計
- 無効化のタイミング
を提案して"
```

判断基準:

| 条件 | キャッシュ向き | キャッシュ不向き |
|------|-------------|---------------|
| 更新頻度 | 低い（1時間以上変わらない） | 高い（秒単位で変わる） |
| 共有可否 | 全ユーザー共通 | ユーザー別 |
| 計算コスト | 重い（JOIN多数、集計等） | 軽い（単純なfindById） |
| 一貫性要件 | 多少のズレが許容できる | リアルタイム性が必要 |

---

## 4-2. Cache-Aside パターンの実装

最も一般的なキャッシュパターン:

```bash
claude "Redis + PrismaのCache-Asideパターンを実装:

対象: GET /products/:id

フロー:
1. キャッシュを確認
2. キャッシュヒット → 返す
3. キャッシュミス → DBから取得 → キャッシュに保存 → 返す

要件:
- TTL: 5分
- キャッシュキー: product:{id}
- シリアライズ: JSON
- 型安全（TypeScript）
- エラー時はキャッシュをスキップしてDBから返す（Failover）"
```

実装例:

```typescript
// src/services/product.service.ts
export class ProductService {
  private readonly CACHE_TTL = 300 // 5分

  constructor(
    private prisma: PrismaClient,
    private redis: Redis
  ) {}

  async getProduct(id: string): Promise<Product | null> {
    const cacheKey = `product:${id}`

    // キャッシュチェック
    try {
      const cached = await this.redis.get(cacheKey)
      if (cached) {
        return JSON.parse(cached) as Product
      }
    } catch (err) {
      // Redisエラーはログだけ記録してDBに fallback
      logger.warn({ err }, 'Redis cache read failed')
    }

    // DBから取得
    const product = await this.prisma.product.findUnique({
      where: { id },
      include: { category: true, images: true }
    })

    // キャッシュに保存（エラーが出ても継続）
    if (product) {
      try {
        await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(product))
      } catch (err) {
        logger.warn({ err }, 'Redis cache write failed')
      }
    }

    return product
  }
}
```

---

## 4-3. キャッシュ無効化戦略

キャッシュの難しさは「いつ無効化するか」だ:

```bash
claude "商品情報が更新された時のキャッシュ無効化を実装:

更新が発生するケース:
- 商品情報の編集（管理画面）
- 在庫数の更新（注文時）
- 価格変更（バッチ処理）

無効化パターン:
1. 直接削除（単一アイテムの更新時）
2. パターン削除（product:* を全て削除）
3. タグベース（category-{id}タグを持つキャッシュを削除）

それぞれの実装と、どのケースにどのパターンが適切かを説明して"
```

---

## 4-4. セッションのRedis管理

```bash
claude "Expressのセッション管理をメモリからRedisに移行:

要件:
- connect-redis を使用
- セッションTTL: 24時間（アクセスごとにリセット）
- セッションID: cryptographically secure random
- セッション固定攻撃の対策
- 本番環境でのhttps-only cookie設定"
```

---

## 4-5. Read-Through / Write-Throughパターン

より高度なキャッシュ戦略:

```bash
claude "Write-Throughキャッシュパターンを実装:

商品の更新時に:
1. DBを更新
2. キャッシュも同時に更新（削除ではなく書き込み）
3. どちらか失敗した場合のリカバリ方法

vs Cache-Aside（更新時削除）との比較:
- メリット/デメリット
- どちらを選ぶべきケース
も合わせて説明して"
```

---

## まとめ

キャッシュ設計でClaude Codeを活用するポイント:

1. **何をキャッシュするか**: 更新頻度・共有可否・計算コストで判断させる
2. **無効化のタイミング**: 更新イベントに合わせて明示的に
3. **エラー時の動作**: Redisが落ちてもDBにfallbackする設計に

次の章では、セキュリティ対策を体系的に実装する。
