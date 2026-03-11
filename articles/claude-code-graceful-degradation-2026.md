---
title: "Claude Codeでグレースフルデグレデーションを設計する：部分的サービス継続・フォールバック階層・ユーザー影響最小化"
emoji: "🛟"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-18 13:00"
---

## はじめに

「推薦エンジンが落ちたら全ページが503になった」——グレースフルデグレデーションで依存サービスが落ちてもコアユーザー体験は維持し、フォールバックで代替コンテンツを提供する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにグレースフルデグレデーション設計ルールを書く

```markdown
## グレースフルデグレデーション設計ルール

### 機能の重要度分類
- CRITICAL: 認証・決済・注文確定 → 絶対に落とさない
- HIGH: 検索・商品一覧 → 最大努力で維持（キャッシュフォールバック）
- MEDIUM: パーソナライズ推薦 → 落ちたら汎用コンテンツで代替
- LOW: リアルタイム閲覧数・ランキング → 落ちたら非表示でOK

### フォールバック階層
1. Primaryサービス（通常通り）
2. Staleキャッシュ（Redisにある古いデータ）
3. 静的フォールバック（ハードコードされたデフォルト）
4. 機能なし（エラーではなく「現在利用できません」UI）

### ユーザーへの通知
- CRITICAL障害: エラーページ（透明性）
- MEDIUM/LOW障害: サイレントデグレード（ユーザーに気づかせない）
- 管理者ダッシュボード: 全デグレード状況をリアルタイム表示
```

---

## グレースフルデグレデーション実装の生成

```
グレースフルデグレデーションシステムを設計してください。

要件：
- 機能重要度分類
- フォールバック階層
- サイレントデグレード
- デグレード状況の追跡

生成ファイル: src/resilience/degradation/
```

---

## 生成されるグレースフルデグレデーション実装

```typescript
// src/resilience/degradation/degradationManager.ts — デグレデーション管理

export type FeaturePriority = 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';

export interface FallbackStrategy<T> {
  staleCache?: () => Promise<T | null>;     // 古いキャッシュから取得
  staticDefault?: () => T;                  // 静的デフォルト値
  gracefulOmit?: true;                      // この機能を省略（nullを返す）
}

export class DegradationManager {
  private readonly degradedFeatures = new Set<string>();

  async execute<T>(
    featureName: string,
    priority: FeaturePriority,
    primary: () => Promise<T>,
    fallback: FallbackStrategy<T>
  ): Promise<T | null> {
    try {
      const result = await primary();

      // 成功: デグレード状態を解除
      if (this.degradedFeatures.has(featureName)) {
        this.degradedFeatures.delete(featureName);
        logger.info({ featureName }, 'Feature recovered from degradation');
        metrics.featureDegraded.set({ feature: featureName }, 0);
      }

      return result;
    } catch (error) {
      return this.handleFailure(featureName, priority, fallback, error as Error);
    }
  }

  private async handleFailure<T>(
    featureName: string,
    priority: FeaturePriority,
    fallback: FallbackStrategy<T>,
    error: Error
  ): Promise<T | null> {
    const isFirstFailure = !this.degradedFeatures.has(featureName);
    this.degradedFeatures.add(featureName);

    if (isFirstFailure) {
      logger.warn({ featureName, priority, error: error.message }, 'Feature degraded');
      metrics.featureDegraded.set({ feature: featureName }, 1);
    }

    // CRITICAL: フォールバックせずエラーをスロー
    if (priority === 'CRITICAL') {
      throw error;
    }

    // フォールバック階層を順に試す

    // 1. Staleキャッシュ
    if (fallback.staleCache) {
      try {
        const cached = await fallback.staleCache();
        if (cached !== null) {
          logger.debug({ featureName }, 'Using stale cache fallback');
          metrics.staleCacheFallback.inc({ feature: featureName });
          return cached;
        }
      } catch (cacheError) {
        logger.debug({ featureName, error: (cacheError as Error).message }, 'Stale cache also failed');
      }
    }

    // 2. 静的デフォルト
    if (fallback.staticDefault) {
      logger.debug({ featureName }, 'Using static default fallback');
      metrics.staticFallback.inc({ feature: featureName });
      return fallback.staticDefault();
    }

    // 3. 機能省略（nullを返す）
    if (fallback.gracefulOmit) {
      logger.debug({ featureName }, 'Gracefully omitting feature');
      return null;
    }

    throw error;
  }

  getDegradedFeatures(): string[] {
    return [...this.degradedFeatures];
  }
}
```

```typescript
// src/resilience/degradation/homepageService.ts — ホームページのデグレデーション例

export class HomePageService {
  private readonly degradation = new DegradationManager();

  async getHomePageData(userId?: string): Promise<HomePageData> {
    const [featured, recommendations, trending, userProfile] = await Promise.all([
      // HIGH: 商品一覧（Staleキャッシュでフォールバック）
      this.degradation.execute(
        'featured-products',
        'HIGH',
        () => productService.getFeaturedProducts(),
        {
          staleCache: () => redis.get('cache:featured-products').then(v => v ? JSON.parse(v) : null),
          staticDefault: () => DEFAULT_FEATURED_PRODUCTS,
        }
      ),

      // MEDIUM: パーソナライズ推薦（汎用推薦でフォールバック）
      this.degradation.execute(
        'personalized-recommendations',
        'MEDIUM',
        () => recommendationService.getForUser(userId),
        {
          staleCache: userId ? () => redis.get(`cache:recs:${userId}`).then(v => v ? JSON.parse(v) : null) : undefined,
          staticDefault: () => POPULAR_PRODUCTS_STATIC,
        }
      ),

      // LOW: トレンドランキング（省略OK）
      this.degradation.execute(
        'trending-ranking',
        'LOW',
        () => analyticsService.getTrending(),
        { gracefulOmit: true }
      ),

      // HIGH: ユーザープロフィール（ログイン中のみ）
      userId ? this.degradation.execute(
        'user-profile',
        'HIGH',
        () => userService.getProfile(userId),
        {
          staleCache: () => redis.get(`cache:profile:${userId}`).then(v => v ? JSON.parse(v) : null),
          gracefulOmit: true,
        }
      ) : Promise.resolve(null),
    ]);

    return {
      featured: featured ?? [],
      recommendations: recommendations ?? [],
      trending: trending ?? null,       // nullの場合はUIで非表示
      userProfile: userProfile ?? null,
      degradedFeatures: this.degradation.getDegradedFeatures(),
    };
  }
}

// 管理者向けデグレード状況API
router.get('/api/admin/degradation/status', requireAdmin, async (req, res) => {
  const [featureFlags, cacheStats] = await Promise.all([
    redis.hGetAll('degradation:flags'),
    redis.info('stats'),
  ]);

  res.json({
    degradedFeatures: degradationManager.getDegradedFeatures(),
    featureFlags,
    timestamp: new Date().toISOString(),
  });
});
```

---

## まとめ

Claude Codeでグレースフルデグレデーションを設計する：

1. **CLAUDE.md** に機能をCRITICAL/HIGH/MEDIUM/LOWに分類・CRITICALはフォールバックなし（エラー）・LOWは省略OKを明記
2. **フォールバック階層（StaleCache→StaticDefault→GracefulOmit）** を順に試す——「推薦エンジンが落ちたら古いキャッシュ→それもなければ人気商品リスト→それもなければ空リスト」と自動的に降格
3. **サイレントデグレード** で`LOW`/`MEDIUM`の障害をユーザーに見せない——推薦が消えてもエラーページにせず、通常UIのまま機能だけ省略して体験を最大化
4. **`getDegradedFeatures()`** で管理者ダッシュボードに現在デグレード中の機能を表示——運用チームがリアルタイムで「どの機能が落ちているか」を確認して対処できる

---

*信頼性設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
