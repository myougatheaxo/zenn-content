---
title: "Claude CodeでStrangler Figパターンを設計する：モノリス段階的移行・トラフィックルーティング・機能フラグ"
emoji: "🌿"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-17 13:00"
---

## はじめに

「モノリスを一気にマイクロサービス化するのはリスクが高い」——Strangler Figパターンで機能を段階的に移行し、旧システムを徐々に「絞り殺し」ながら新システムへ安全に切り替える設計をClaude Codeに生成させる。

---

## CLAUDE.mdにStrangler Fig設計ルールを書く

```markdown
## Strangler Figパターン設計ルール

### 移行戦略
- Facadeレイヤーが全リクエストを受け取り、新旧どちらかにルーティング
- 機能フラグ（feature flag）で移行率を0%→100%に段階的に上げる
- 新サービスの応答をシャドーモードで検証してから切り替える

### データ移行
- 新旧DBは同期期間中は二重書き込み
- 移行完了後に旧DBへの書き込みを停止→旧テーブル削除
- データ不整合検出: 定期的に新旧の差分チェック

### ロールバック
- 機能フラグを0%に戻せば即座にモノリスに切り戻し
- 移行完了宣言前はロールバックパスを常に維持
```

---

## Strangler Fig実装の生成

```
Strangler Figパターンを設計してください。

要件：
- ルーティングFacade
- 機能フラグ制御
- シャドーモード検証
- データ二重書き込み

生成ファイル: src/migration/stranglerFig/
```

---

## 生成されるStrangler Fig実装

```typescript
// src/migration/stranglerFig/migrationFacade.ts — 移行Facade

export interface ServiceAdapter<TRequest, TResponse> {
  name: string;
  handle: (request: TRequest) => Promise<TResponse>;
}

export class StranglerFigFacade<TRequest, TResponse> {
  constructor(
    private readonly featureName: string,
    private readonly legacyService: ServiceAdapter<TRequest, TResponse>,
    private readonly newService: ServiceAdapter<TRequest, TResponse>
  ) {}

  async handle(request: TRequest, userId?: string): Promise<TResponse> {
    const rolloutPercent = await this.getRolloutPercent();

    // シャドーモード: 新サービスを並行実行して結果を比較（本番には影響しない）
    if (rolloutPercent === 0) {
      return this.shadowMode(request);
    }

    // ユーザーベースの段階的ルーティング
    const useNewService = userId
      ? this.isUserInRollout(userId, rolloutPercent)
      : Math.random() * 100 < rolloutPercent;

    if (useNewService) {
      return this.handleWithFallback(request, this.newService, this.legacyService);
    } else {
      return this.legacyService.handle(request);
    }
  }

  // シャドーモード: 旧サービスの結果を返しながら新サービスも非同期実行して差分を記録
  private async shadowMode(request: TRequest): Promise<TResponse> {
    const legacyResult = await this.legacyService.handle(request);

    // 非同期でシャドー実行（本番レスポンスに影響しない）
    setImmediate(async () => {
      try {
        const newResult = await this.newService.handle(request);
        const isDiff = JSON.stringify(legacyResult) !== JSON.stringify(newResult);

        await prisma.shadowComparisonLog.create({
          data: {
            featureName: this.featureName,
            request: JSON.stringify(request),
            legacyResult: JSON.stringify(legacyResult),
            newResult: JSON.stringify(newResult),
            hasDifference: isDiff,
            comparedAt: new Date(),
          },
        });

        if (isDiff) {
          logger.warn({ featureName: this.featureName }, 'Shadow mode: response difference detected');
          metrics.shadowDiffRate.inc({ feature: this.featureName });
        }
      } catch (error) {
        logger.error({ featureName: this.featureName, error }, 'Shadow mode: new service error');
      }
    });

    return legacyResult;
  }

  private async handleWithFallback(
    request: TRequest,
    primary: ServiceAdapter<TRequest, TResponse>,
    fallback: ServiceAdapter<TRequest, TResponse>
  ): Promise<TResponse> {
    try {
      const result = await primary.handle(request);
      metrics.migrationSuccess.inc({ feature: this.featureName, service: primary.name });
      return result;
    } catch (error) {
      logger.error({ feature: this.featureName, error }, 'New service failed, falling back to legacy');
      metrics.migrationFallback.inc({ feature: this.featureName });
      return fallback.handle(request);
    }
  }

  private isUserInRollout(userId: string, percent: number): boolean {
    // ユーザーIDのハッシュで一貫したルーティング（同じユーザーは常に同じサービス）
    const hash = Array.from(userId).reduce((acc, c) => acc + c.charCodeAt(0), 0);
    return (hash % 100) < percent;
  }

  private async getRolloutPercent(): Promise<number> {
    const cached = await redis.get(`migration:rollout:${this.featureName}`);
    if (cached !== null) return parseFloat(cached);

    const flag = await prisma.featureFlag.findUnique({ where: { name: this.featureName } });
    const percent = flag?.rolloutPercent ?? 0;

    await redis.set(`migration:rollout:${this.featureName}`, percent.toString(), { EX: 60 });
    return percent;
  }
}
```

```typescript
// src/migration/stranglerFig/dualWriteRepository.ts — データ二重書き込み

export class DualWriteUserRepository {
  constructor(
    private readonly legacyDb: PrismaClient,
    private readonly newDb: NewUserRepository
  ) {}

  // 書き込みは両方に実施（整合性を維持）
  async createUser(data: CreateUserInput): Promise<User> {
    // 新DBをプライマリとして先に書き込み
    const newUser = await this.newDb.create(data);

    // 旧DBにも書き込み（フォールバック用）
    try {
      await this.legacyDb.user.create({
        data: {
          id: newUser.id,  // 同じIDを使用（ID空間を統一）
          email: data.email,
          name: data.name,
          createdAt: newUser.createdAt,
        },
      });
    } catch (error) {
      // 旧DBへの書き込み失敗は警告のみ（新DBが正）
      logger.warn({ userId: newUser.id, error }, 'Dual-write: legacy DB write failed');
      metrics.dualWriteFailure.inc({ entity: 'user', direction: 'legacy' });
    }

    return newUser;
  }

  // 読み込みはFeatureFlagに従ってルーティング
  async findById(id: string): Promise<User | null> {
    const useNew = await this.isNewServiceActive(id);

    if (useNew) {
      return this.newDb.findById(id);
    } else {
      return this.legacyDb.user.findUnique({ where: { id } });
    }
  }

  // 整合性チェック（バッチで定期実行）
  async verifyConsistency(limit = 1000): Promise<ConsistencyReport> {
    const newUsers = await this.newDb.findMany({ limit, orderBy: { createdAt: 'desc' } });
    const diffs: string[] = [];

    for (const newUser of newUsers) {
      const legacyUser = await this.legacyDb.user.findUnique({ where: { id: newUser.id } });

      if (!legacyUser) {
        diffs.push(`User ${newUser.id}: exists in new DB but not in legacy`);
      } else if (legacyUser.email !== newUser.email) {
        diffs.push(`User ${newUser.id}: email mismatch (new: ${newUser.email}, legacy: ${legacyUser.email})`);
      }
    }

    return { checked: newUsers.length, differences: diffs, isConsistent: diffs.length === 0 };
  }
}

// 管理者API: ロールアウト率を変更
router.put('/api/admin/migration/:featureName/rollout', requireAdmin, async (req, res) => {
  const { percent } = req.body; // 0-100
  await prisma.featureFlag.upsert({
    where: { name: req.params.featureName },
    create: { name: req.params.featureName, rolloutPercent: percent },
    update: { rolloutPercent: percent },
  });
  await redis.del(`migration:rollout:${req.params.featureName}`);
  res.json({ message: `Rollout set to ${percent}%` });
});
```

---

## まとめ

Claude CodeでStrangler Figパターンを設計する：

1. **CLAUDE.md** にFacadeレイヤーで新旧にルーティング・機能フラグで0%→100%段階移行・シャドーモードで事前検証・二重書き込みで整合性維持を明記
2. **シャドーモード** で旧サービスの結果を返しながら新サービスを非同期実行——差分ログで「新旧の応答が一致するか」を本番トラフィックで検証してから切り替え
3. **ユーザーIDハッシュルーティング** で同じユーザーは常に同じサービスに当たる——「AさんはA見えてBさんはB見える」を防止し、A/Bテストと同じ安定性を確保
4. **整合性チェックバッチ** で新旧DBの差分を定期検出——移行完了宣言前に「旧DBに書いたデータが新DBにもあるか」を自動確認

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
