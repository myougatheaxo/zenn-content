---
title: "Claude Codeでフィーチャーフラグ戦略を設計する：段階的ロールアウト・Kill Switch・実験管理"
emoji: "🚦"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "devops"]
published: true
published_at: "2026-03-13 13:00"
---

## はじめに

デプロイとリリースを分離する——フィーチャーフラグで新機能を安全に段階ロールアウトし、問題があればKill Switchで即座にオフにする。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにフィーチャーフラグ設計ルールを書く

```markdown
## フィーチャーフラグ設計ルール

### フラグ種別
- Kill Switch: 本番障害時の緊急オフ
- Rollout: 段階ロールアウト（1%→5%→20%→100%）
- Experiment: A/Bテスト（指定期間で自動終了）
- Entitlement: プラン別機能解放

### 評価ルール
- ユーザーIDのsha256ハッシュで決定論的に振り分け
- 同じユーザーは常に同じフラグ値を受け取る
- 優先度: Kill Switch > Entitlement > Rollout > Default

### ライフサイクル管理
- 実験フラグは30日で自動アーカイブ
- デッドフラグはコードから削除（技術的負債防止）
- フラグ変更は全てAuditログに記録
```

---

## フィーチャーフラグシステムの生成

```
フィーチャーフラグ管理システムを設計してください。

要件：
- 複数フラグ種別（Kill Switch・Rollout・Experiment）
- ユーザーセグメント指定
- Kill Switch（緊急オフ）
- Redisキャッシュ
- フラグ変更時の即時反映

生成ファイル: src/features/
```

---

## 生成されるフィーチャーフラグ実装

```typescript
// src/features/featureFlag.ts

type FlagType = 'kill_switch' | 'rollout' | 'experiment' | 'entitlement';

interface FeatureFlag {
  key: string;
  type: FlagType;
  enabled: boolean;
  rolloutPercentage?: number; // 0-100
  allowedUserIds?: string[];  // ホワイトリスト
  allowedPlans?: string[];    // 'pro', 'enterprise'
  expiresAt?: Date;           // 実験フラグの終了日
}

export class FeatureFlagService {
  // フラグ評価（決定論的ハッシュベース）
  async isEnabled(
    flagKey: string,
    context: { userId?: string; planId?: string }
  ): Promise<boolean> {
    // Redisキャッシュを確認（1分TTL）
    const cached = await redis.get(`flag:${flagKey}`);
    const flag: FeatureFlag = cached
      ? JSON.parse(cached)
      : await this.loadAndCacheFlag(flagKey);

    if (!flag) return false; // 未定義フラグはオフ

    // Kill Switch: 即座にオフ
    if (flag.type === 'kill_switch' && !flag.enabled) return false;

    // フラグ全体が無効
    if (!flag.enabled) return false;

    // 実験フラグの期限切れ確認
    if (flag.expiresAt && new Date() > flag.expiresAt) return false;

    // ホワイトリスト（強制有効）
    if (context.userId && flag.allowedUserIds?.includes(context.userId)) return true;

    // Entitlement（プラン制限）
    if (flag.type === 'entitlement' && flag.allowedPlans) {
      return !!context.planId && flag.allowedPlans.includes(context.planId);
    }

    // Rollout（段階ロールアウト）
    if (flag.rolloutPercentage !== undefined && context.userId) {
      const bucket = await this.getUserBucket(context.userId, flagKey);
      return bucket < flag.rolloutPercentage;
    }

    return flag.enabled;
  }

  // SHA256ハッシュでユーザーを0-99のバケットに割り当て（決定論的）
  private async getUserBucket(userId: string, flagKey: string): Promise<number> {
    const hash = createHash('sha256')
      .update(`${userId}:${flagKey}`)
      .digest('hex');

    // 16進数の最初の8文字を整数に変換して0-99に正規化
    const numericHash = parseInt(hash.slice(0, 8), 16);
    return numericHash % 100;
  }

  private async loadAndCacheFlag(flagKey: string): Promise<FeatureFlag | null> {
    const flag = await prisma.featureFlag.findUnique({ where: { key: flagKey } });
    if (flag) {
      await redis.set(`flag:${flagKey}`, JSON.stringify(flag), { EX: 60 }); // 1分キャッシュ
    }
    return flag;
  }

  // Kill Switch（即座に全ユーザーオフ）
  async killSwitch(flagKey: string, reason: string, operatorId: string): Promise<void> {
    await prisma.$transaction([
      prisma.featureFlag.update({
        where: { key: flagKey },
        data: { enabled: false },
      }),
      prisma.featureFlagAudit.create({
        data: { flagKey, action: 'kill_switch', reason, operatorId, changedAt: new Date() },
      }),
    ]);

    // Redisキャッシュを即座に無効化
    await redis.del(`flag:${flagKey}`);

    logger.warn({ flagKey, reason, operatorId }, 'Feature flag kill switch activated');
    await sendSlackAlert(`🛑 Kill Switch: ${flagKey} disabled by ${operatorId}. Reason: ${reason}`);
  }

  // 段階ロールアウト（1%→5%→20%→100%）
  async progressRollout(flagKey: string, percentage: number, operatorId: string): Promise<void> {
    const current = await prisma.featureFlag.findUnique({ where: { key: flagKey } });
    const prev = current?.rolloutPercentage ?? 0;

    // ロールアウトの逆行は禁止（誤操作防止）
    if (percentage < prev) {
      throw new ValidationError(`Cannot reduce rollout from ${prev}% to ${percentage}%`);
    }

    await prisma.$transaction([
      prisma.featureFlag.update({
        where: { key: flagKey },
        data: { rolloutPercentage: percentage },
      }),
      prisma.featureFlagAudit.create({
        data: {
          flagKey,
          action: 'rollout_progress',
          details: { from: prev, to: percentage },
          operatorId,
          changedAt: new Date(),
        },
      }),
    ]);

    await redis.del(`flag:${flagKey}`); // キャッシュ無効化
    logger.info({ flagKey, from: prev, to: percentage }, 'Rollout progressed');
  }
}
```

```typescript
// src/features/middleware.ts — リクエストコンテキスト統合

export const featureFlags = new FeatureFlagService();

// 使用例: 新UI
router.get('/dashboard', requireAuth, async (req, res) => {
  const showNewUI = await featureFlags.isEnabled('new_dashboard_ui', {
    userId: req.user.id,
    planId: req.user.planId,
  });

  if (showNewUI) {
    return res.render('dashboard-v2');
  }
  return res.render('dashboard');
});

// React Hook（フロントエンド）
export function useFeatureFlag(flagKey: string): boolean {
  const { user } = useAuth();
  const [enabled, setEnabled] = useState(false);

  useEffect(() => {
    fetch(`/api/features/${flagKey}?userId=${user.id}`)
      .then(r => r.json())
      .then(data => setEnabled(data.enabled));
  }, [flagKey, user.id]);

  return enabled;
}
```

---

## まとめ

Claude Codeでフィーチャーフラグを設計する：

1. **CLAUDE.md** に4種別・ハッシュベース決定論的割り当て・Kill Switch即時無効・Audit必須を明記
2. **SHA256ハッシュ** でユーザーIDとフラグキーの組み合わせからバケット決定（同じユーザーは常に同じ結果）
3. **Redis 1分キャッシュ** でフラグ評価を高速化——Kill Switch時はキャッシュを即座に削除
4. **ロールアウト逆行防止** で誤操作を検知（1%→5%→20%→100%の一方向性）

---

*フィーチャーフラグ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
