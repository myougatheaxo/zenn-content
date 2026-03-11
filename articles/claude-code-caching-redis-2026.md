---
title: "Claude Codeでキャッシュ戦略を実装する：Redis・無効化・グレースフル設計"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "redis", "caching", "nodejs", "パフォーマンス"]
published: true
---

## はじめに

キャッシュの設計ミスは二種類ある。キャッシュしすぎて古いデータを返す問題と、Redisが落ちたらサービスも落ちる問題だ。

CLAUDE.mdでキャッシュパターンを定義し、Claude Codeが正しい実装を生成するようにする。

---

## CLAUDE.mdにキャッシュルールを書く

```markdown
## キャッシュ設計ルール

### キャッシュ対象とTTL
- マスタデータ（設定・ロール・フラグ）: 5分
- 重い計算（集計・レポート）: 15分
- 外部APIレスポンス: 1分
- ユーザーセッション: セッション有効期間

### キャッシュ非対象
- 書き込み直後に読まれるデータ
- リアルタイムデータ（在庫・通知等）
- 決済中のトランザクション

### キャッシュキー命名
- 形式: `{service}:{entity}:{id}` または `{service}:{entity}:list:{hash}`
- ユーザー入力をキーに直接使わない（インジェクションリスク）
- 例: `user:profile:123`, `config:features:v1`

### Redisエラー時の動作（必須）
- エラーはキャッシュミスとして扱う（リクエストを失敗させない）
- エラーはlogger.tsで記録するがthrowしない
- グレースフル・デグラデーション: Redisなしでも機能する設計

### 無効化戦略
- TTLベース: 一貫性が柔軟な場合
- イベントベース: 重要データの変更時（ドメインイベント経由）
- 一括無効化: Redis SCANを使う（KEYSコマンドは本番禁止）
```

---

## キャッシュ・アサイドパターンの生成

```
getUserProfile()にキャッシュ・アサイドパターンを実装してください。

CLAUDE.mdのキャッシュルールに従って：
1. Redisでキャッシュを確認
2. ヒット → キャッシュデータを返す
3. ミス → DBから取得 → Redisに保存（TTL 5分）→ 返す
4. Redisエラー → DBから取得してグレースフルに機能させる

キャッシュキー: user:profile:{userId}
```

生成コード：

```typescript
async function getUserProfile(userId: string): Promise<UserProfile | null> {
  const cacheKey = `user:profile:${userId}`;

  // キャッシュ確認（エラー時はnull）
  const cached = await cache.get<UserProfile>(cacheKey);
  if (cached) return cached;

  // DBから取得
  const profile = await db.user.findUnique({ where: { id: userId } });
  if (!profile) return null;

  // 非同期でキャッシュ保存（リクエストをブロックしない）
  cache.set(cacheKey, profile, 300).catch(logger.error);

  return profile;
}
```

---

## キャッシュユーティリティモジュールの生成

```
Redisキャッシュユーティリティを生成してください。

要件：
- get<T>(key): T | null（エラー時はnull）
- set(key, value, ttlSeconds): Promise<void>（エラー時は無視）
- del(key | key[]): Promise<void>
- Redis障害時にリクエストを失敗させない
- エラーはlogger.tsで記録

保存先：src/lib/cache.ts
```

---

## キャッシュ無効化パターン

ユーザー更新時のキャッシュ無効化：

```
ユーザーがプロフィールを更新した際のキャッシュ無効化を実装してください。

要件：
- user:profile:{userId} を削除
- user:list:* の全キーをSCAN+DELで削除（KEYSコマンド禁止）
- 直接cacheを呼ぶのではなく、ドメインイベント('user.updated')をemitする設計
```

---

## キャッシュのテスト生成

```
キャッシュ付きgetUserProfile()のテストを生成してください。

テストケース：
1. キャッシュヒット: DBを呼ばずキャッシュデータを返す
2. キャッシュミス: DBから取得→キャッシュ保存→返す
3. ユーザーなし: nullを返す（キャッシュに保存しない）
4. Redisエラー: DBフォールバックで正常動作する

RedisとPrismaの両方をモック。
```

---

## まとめ

Claude Codeでキャッシュを安全に設計する：

1. **CLAUDE.md** にキャッシュ対象・TTL・エラー処理を定義
2. **グレースフル**: Redisエラー時もリクエストを失敗させない
3. **無効化**: ドメインイベント経由で整理
4. **一括削除**: SCANを使う（KEYSは本番禁止）

---

*キャッシュ設計の問題を自動検出するには **Code Review Pack（¥980）** を使えます。PromptWorksで購入できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
