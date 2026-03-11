---
title: "第3章 データベース最適化 — スキーマ設計からN+1解消まで"
---

# 第3章 データベース最適化 — スキーマ設計からN+1解消まで

DBのパフォーマンス問題はプロダクションに出るまで気づかないことが多い。Claude Codeを使えば、設計段階で問題を潰せる。

---

## 3-1. スキーマ設計のレビュー

```bash
claude "prisma/schema.prisma の設計をレビューして:

1. 正規化の問題（重複データ、更新時の不整合リスク）
2. インデックスの過不足
   - WHERE句に使われているがインデックスがないカラム
   - 不要なインデックス（書き込みコストを増やすだけのもの）
3. 外部キー制約の漏れ
4. NULL許可の設計（NOT NULL にすべきカラム）
5. データ型の適切さ（VARCHAR(255) vs TEXT、INT vs BIGINT等）"
```

よくある問題:

```prisma
// ❌ インデックスが足りない
model Order {
  id        String   @id
  userId    String   // WHERE userId でよく検索するが index なし
  status    String   // WHERE status でよく絞り込むが index なし
  createdAt DateTime
}

// ✅ 適切なインデックス
model Order {
  id        String   @id
  userId    String
  status    String
  createdAt DateTime

  @@index([userId])
  @@index([status, createdAt])  // 複合インデックス（順序が重要）
}
```

---

## 3-2. N+1クエリの検出と解消

N+1はORMを使うと自然に発生する。Claude Codeに検出させる:

```bash
claude "src/services/ のコードでN+1クエリが発生している箇所を特定して。

典型的なパターン:
- forループ内でDBクエリ（findById等）を実行している
- リレーションを都度フェッチしている

検出した箇所は、Prismaのinclude/selectで解消する修正案も提示して"
```

問題のパターンと修正:

```typescript
// ❌ N+1問題
const orders = await prisma.order.findMany()
for (const order of orders) {
  const user = await prisma.user.findUnique({ where: { id: order.userId } })
  // N件のorderに対してN回のクエリ発生
}

// ✅ N+1解消（JOINで1回のクエリ）
const orders = await prisma.order.findMany({
  include: {
    user: {
      select: { id: true, name: true, email: true }
    }
  }
})
```

---

## 3-3. クエリパフォーマンスの分析

```bash
claude "以下のクエリのパフォーマンスを分析して改善案を提示:

現在のクエリ:
SELECT u.*, o.total, p.name
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
LEFT JOIN products p ON o.product_id = p.id
WHERE u.status = 'active'
AND o.created_at > NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC

考慮すること:
- 必要なインデックス
- クエリの実行計画（EXPLAIN ANALYZE）の予測
- ページネーションの追加
- 取得カラムの絞り込み"
```

---

## 3-4. マイグレーション戦略

```bash
claude "本番DBへの安全なマイグレーション戦略を立案:

変更内容: usersテーブルにfull_nameカラムを追加
（既存の first_name + last_name から生成）

要件:
- ダウンタイムゼロ
- ロールバック可能
- 既存データの移行

手順:
1. カラム追加（NULL許可）
2. バックグラウンドジョブで既存データを埋める
3. アプリケーションを両フィールド対応に更新
4. カラムをNOT NULLに変更
5. 旧カラムを削除（最終段階）

Prismaマイグレーションファイルとして生成して"
```

---

## 3-5. 接続プールの最適化

```bash
claude "PrismaのDB接続プール設定を最適化:

環境:
- アプリサーバー: 3インスタンス
- DBサーバー: PostgreSQL 16、最大接続数300
- 予想リクエスト数: 1000 req/min

最適な接続プール設定と根拠を説明して。
pool_timeout, connection_limit, pool_mode (session vs transaction)についても"
```

---

## まとめ

DBパフォーマンス改善でClaude Codeが役立つ場面:

| タスク | プロンプト例 |
|--------|------------|
| スキーマレビュー | 「schema.prismaのインデックス設計をレビュー」 |
| N+1検出 | 「ループ内DBクエリを探して修正案を提示」 |
| クエリ最適化 | 「このSQLのEXPLAIN ANALYZEを予測して改善案を」 |
| マイグレーション | 「ゼロダウンタイムでカラム追加する手順を」 |

次の章では、キャッシュ設計でDBの負荷をさらに下げる。
