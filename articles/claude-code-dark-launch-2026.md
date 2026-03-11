---
title: "Claude Codeでダークローンチを設計する：トラフィックシャドウイング・段階的ロールアウト"
emoji: "🌙"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "devops"]
published: true
published_at: "2026-03-16 10:00"
---

## はじめに

「新機能を本番でいきなり全ユーザーに公開するのが怖い」——ダークローンチ（Dark Launch）で本番トラフィックをシャドウイングし、実データで新機能を検証してから段階的に展開する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにダークローンチ設計ルールを書く

```markdown
## ダークローンチ設計ルール

### シャドウイング方式
- 本番リクエストを新旧両方の実装に送る（応答は旧実装を使用）
- 新実装の応答・エラー・レイテンシーをRedisに記録
- 差異をレポートして問題がないことを確認してからロールアウト

### 段階的ロールアウト
- 0% → 1% → 5% → 10% → 25% → 50% → 100% の段階で展開
- 各段階で1時間以上観察してエラー率・レイテンシーを確認
- 問題検出時は即座に0%に戻す（フラグOFF）

### ユーザーセグメント
- 内部ユーザー（@company.com）: 最初のロールアウト対象
- βユーザー: 自発的に新機能を有効化
- 段階的ロールアウト: ユーザーIDのhashで決定的に割り振り
```

---

## ダークローンチ実装の生成

```
ダークローンチ・トラフィックシャドウイングシステムを設計してください。

要件：
- リクエストシャドウイング（非同期）
- 段階的ロールアウト（1%→100%）
- 差異検出・メトリクス記録
- 即時ロールバック機能

生成ファイル: src/launch/darkLaunch/
```

---

## 生成されるダークローンチ実装

```typescript
// src/launch/darkLaunch/rolloutConfig.ts — ロールアウト設定管理

export interface RolloutConfig {
  featureKey: string;
  percentage: number;      // 0-100
  enabledForInternal: boolean; // 内部ユーザー(@company.com)は常時有効
  enabledUserIds?: string[];   // 特定ユーザーに固定で有効
  disabledUserIds?: string[];  // 特定ユーザーに固定で無効（問題報告者等）
  shadowingOnly: boolean;      // trueの場合は旧実装を使用（シャドウイングのみ）
}

export class RolloutManager {
  private readonly configPrefix = 'rollout:config';

  async getConfig(featureKey: string): Promise<RolloutConfig | null> {
    const cached = await redis.get(`${this.configPrefix}:${featureKey}`);
    return cached ? JSON.parse(cached) : null;
  }

  async setConfig(config: RolloutConfig): Promise<void> {
    await redis.set(
      `${this.configPrefix}:${config.featureKey}`,
      JSON.stringify(config),
      { EX: 3600 } // 1時間キャッシュ（変更後はflushが必要）
    );
    logger.info({ config }, 'Rollout config updated');
  }

  // ユーザーが新機能を受け取るか判定
  isUserInRollout(userId: string, userEmail: string, config: RolloutConfig): boolean {
    // 強制無効
    if (config.disabledUserIds?.includes(userId)) return false;

    // 強制有効
    if (config.enabledUserIds?.includes(userId)) return true;

    // 内部ユーザー
    if (config.enabledForInternal && userEmail.endsWith('@company.com')) return true;

    // パーセンテージロールアウト（ユーザーIDから決定的に計算）
    const hash = this.hashUserId(userId);
    return (hash % 100) < config.percentage;
  }

  private hashUserId(userId: string): number {
    // djb2ハッシュ（高速・均一分布）
    let hash = 5381;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) + hash) + userId.charCodeAt(i);
      hash = hash & hash; // 32bit integer
    }
    return Math.abs(hash);
  }
}
```

```typescript
// src/launch/darkLaunch/shadowRunner.ts — シャドウイング実行

interface ShadowResult {
  featureKey: string;
  requestId: string;
  oldResult: unknown;
  newResult?: unknown;
  newError?: string;
  oldLatencyMs: number;
  newLatencyMs?: number;
  hasDiff: boolean;
}

export class ShadowRunner {
  // シャドウイング: 本番は旧実装、バックグラウンドで新実装も実行
  async runWithShadowing<T>(
    featureKey: string,
    oldFn: () => Promise<T>,
    newFn: () => Promise<T>,
    requestId: string
  ): Promise<T> {
    const oldStart = Date.now();
    const oldResult = await oldFn(); // 本番レスポンス（ブロッキング）
    const oldLatencyMs = Date.now() - oldStart;

    // 新実装は非同期でシャドウ実行（レスポンスには使わない）
    setImmediate(async () => {
      const newStart = Date.now();
      let newResult: T | undefined;
      let newError: string | undefined;

      try {
        newResult = await newFn();
      } catch (err) {
        newError = (err as Error).message;
      }

      const newLatencyMs = Date.now() - newStart;
      const hasDiff = newError !== undefined || !deepEqual(oldResult, newResult);

      const shadowResult: ShadowResult = {
        featureKey,
        requestId,
        oldResult,
        newResult,
        newError,
        oldLatencyMs,
        newLatencyMs,
        hasDiff,
      };

      await this.recordShadowResult(shadowResult);
    });

    return oldResult;
  }

  private async recordShadowResult(result: ShadowResult): Promise<void> {
    const key = `shadow:${result.featureKey}`;
    const pipeline = redis.pipeline();

    // 差異率の追跡
    pipeline.incr(`${key}:total`);
    pipeline.expire(`${key}:total`, 86400);
    if (result.hasDiff) {
      pipeline.incr(`${key}:diff`);
      pipeline.expire(`${key}:diff`, 86400);
    }

    // エラーサンプル保存（差異がある場合のみ、最大100件）
    if (result.hasDiff) {
      pipeline.lpush(`${key}:samples`, JSON.stringify({
        requestId: result.requestId,
        newError: result.newError,
        diffSummary: result.newError ? 'error' : 'value_diff',
        timestamp: Date.now(),
      }));
      pipeline.ltrim(`${key}:samples`, 0, 99); // 最大100件
    }

    await pipeline.exec();
  }

  async getDiffRate(featureKey: string): Promise<{ diffRate: number; total: number; diffs: number }> {
    const [total, diffs] = await Promise.all([
      redis.get(`shadow:${featureKey}:total`).then(v => parseInt(v ?? '0')),
      redis.get(`shadow:${featureKey}:diff`).then(v => parseInt(v ?? '0')),
    ]);
    return { total, diffs, diffRate: total > 0 ? diffs / total : 0 };
  }
}
```

```typescript
// src/launch/darkLaunch/darkLaunchMiddleware.ts — ミドルウェア統合

const rolloutManager = new RolloutManager();
const shadowRunner = new ShadowRunner();

export function withDarkLaunch<T>(featureKey: string) {
  return async (
    req: Request,
    oldFn: () => Promise<T>,
    newFn: () => Promise<T>
  ): Promise<T> => {
    const config = await rolloutManager.getConfig(featureKey);

    if (!config) return oldFn(); // 設定なし: 旧実装

    const userId = req.userId ?? 'anonymous';
    const userEmail = req.user?.email ?? '';
    const inRollout = rolloutManager.isUserInRollout(userId, userEmail, config);

    if (config.shadowingOnly) {
      // シャドウイングのみ: 旧実装を返しつつ新実装もバックグラウンド実行
      return shadowRunner.runWithShadowing(featureKey, oldFn, newFn, req.requestId);
    }

    if (inRollout) {
      return newFn(); // 新実装を使用
    } else {
      return oldFn(); // 旧実装を使用
    }
  };
}

// 使用例: 検索APIの新実装をダークローンチ
router.get('/api/search', requireAuth, async (req, res) => {
  const launch = withDarkLaunch<SearchResult[]>('new-search-engine');

  const results = await launch(
    req,
    () => oldSearchService.search(req.query.q as string),  // 旧実装
    () => newSearchService.search(req.query.q as string),  // 新実装
  );

  res.json(results);
});

// ダークローンチ管理API
router.get('/admin/dark-launch/:featureKey/metrics', requireAdmin, async (req, res) => {
  const { featureKey } = req.params;
  const metrics = await shadowRunner.getDiffRate(featureKey);
  const samples = await redis.lrange(`shadow:${featureKey}:samples`, 0, 9);

  res.json({
    ...metrics,
    recentDiffs: samples.map(s => JSON.parse(s)),
  });
});

router.put('/admin/dark-launch/:featureKey', requireAdmin, async (req, res) => {
  await rolloutManager.setConfig({ featureKey: req.params.featureKey, ...req.body });
  res.json({ success: true });
});
```

---

## まとめ

Claude Codeでダークローンチを設計する：

1. **CLAUDE.md** に段階ロールアウト（0→1→5→10→25→50→100%）・ユーザーIDハッシュで決定的割り振り・即時ロールバック方針を明記
2. **シャドウイング** で本番トラフィックを旧実装で処理しつつ新実装をsetImmediateで並行実行——ユーザー体験に影響なく新実装を検証
3. **差異率トラッキング** でRedisのtotal/diff比率を1日ウィンドウで監視——差異サンプルを100件まで保存して差異の内容を調査
4. **管理APIで%を操作** ——問題発見時はpercentage:0に設定するだけで即座にロールバック完了

---

*デプロイ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
