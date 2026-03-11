---
title: "Claude CodeでA/Bテストを設計する：フィーチャーフラグ・実験管理・統計的有意性"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "testing", "analytics"]
published: true
---

## はじめに

「新UIと旧UIどちらが良いか？」——A/Bテストなしに主観で決めると間違える。フィーチャーフラグで安全に実験を管理し、統計的有意性を検証する。Claude Codeに設計を生成させる。

---

## CLAUDE.mdにA/Bテスト設計ルールを書く

```markdown
## A/Bテスト・フィーチャーフラグ設計ルール

### フィーチャーフラグ
- 環境変数ではなくDBで管理（動的切り替え可能）
- ユーザーIDでコホート分割（一貫したUX）
- フラグのキャッシュTTL: 60秒（DB負荷対策）

### 実験管理
- 1実験 = コントロール + バリアント（複数バリアント可）
- サンプルサイズを事前計算（最低1000ユーザー/バリアント）
- 実験期間: 最低2週間（週次トラフィック変動を吸収）

### 統計的有意性
- p値 < 0.05 で結果を有意とする
- 実験開始から7日未満の結果は参照しない（novelty effect）
- 複数指標同時検証はBonferroni補正を適用
```

---

## A/Bテストシステムの生成

```
フィーチャーフラグとA/Bテストシステムを設計してください。

要件：
- ユーザーIDベースでバリアント割り当て（一貫性）
- DBでフラグ管理（動的変更可能）
- イベントトラッキング（コンバージョン記録）
- 統計的有意性の判定

生成ファイル: src/experiments/abTest.ts
```

---

## 生成されるA/Bテスト実装

```typescript
// src/experiments/abTest.ts
import crypto from 'crypto';

interface Experiment {
  id: string;
  name: string;
  variants: Variant[];
  startedAt: Date;
  endedAt: Date | null;
  status: 'running' | 'paused' | 'completed';
}

interface Variant {
  id: string;
  name: string;
  weight: number; // 0-100
}

interface Assignment {
  experimentId: string;
  variantId: string;
  variantName: string;
}

// ユーザーIDとexperimentIDから決定論的にバリアントを割り当て
function assignVariant(userId: string, experiment: Experiment): Assignment {
  // SHA256ハッシュで0-99の整数を生成（一貫性のある割り当て）
  const hash = crypto
    .createHash('sha256')
    .update(`${userId}:${experiment.id}`)
    .digest('hex');

  const bucket = parseInt(hash.slice(0, 8), 16) % 100;

  let cumulative = 0;
  for (const variant of experiment.variants) {
    cumulative += variant.weight;
    if (bucket < cumulative) {
      return {
        experimentId: experiment.id,
        variantId: variant.id,
        variantName: variant.name,
      };
    }
  }

  // フォールバック（合計weightが100に満たない場合）
  const lastVariant = experiment.variants[experiment.variants.length - 1];
  return {
    experimentId: experiment.id,
    variantId: lastVariant.id,
    variantName: lastVariant.name,
  };
}

export class ABTestService {
  private cache = new Map<string, { experiment: Experiment; expiresAt: number }>();

  async getExperiment(experimentName: string): Promise<Experiment | null> {
    // キャッシュチェック（60秒TTL）
    const cached = this.cache.get(experimentName);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.experiment;
    }

    const experiment = await prisma.experiment.findFirst({
      where: { name: experimentName, status: 'running' },
      include: { variants: true },
    });

    if (experiment) {
      this.cache.set(experimentName, {
        experiment,
        expiresAt: Date.now() + 60_000, // 60秒キャッシュ
      });
    }

    return experiment;
  }

  async getVariant(userId: string, experimentName: string): Promise<Assignment | null> {
    const experiment = await this.getExperiment(experimentName);
    if (!experiment) return null;

    // 既存の割り当てがあれば再利用
    const existing = await prisma.experimentAssignment.findUnique({
      where: {
        userId_experimentId: { userId, experimentId: experiment.id },
      },
    });

    if (existing) {
      return {
        experimentId: experiment.id,
        variantId: existing.variantId,
        variantName: existing.variant.name,
      };
    }

    // 新規割り当て（決定論的アルゴリズム）
    const assignment = assignVariant(userId, experiment);

    await prisma.experimentAssignment.create({
      data: {
        userId,
        experimentId: experiment.id,
        variantId: assignment.variantId,
        assignedAt: new Date(),
      },
    });

    return assignment;
  }

  // コンバージョンイベントを記録
  async trackConversion(
    userId: string,
    experimentName: string,
    eventName: string,
    value?: number
  ): Promise<void> {
    const experiment = await this.getExperiment(experimentName);
    if (!experiment) return;

    const assignment = await prisma.experimentAssignment.findUnique({
      where: {
        userId_experimentId: { userId, experimentId: experiment.id },
      },
    });

    if (!assignment) return;

    await prisma.experimentEvent.create({
      data: {
        experimentId: experiment.id,
        variantId: assignment.variantId,
        userId,
        eventName,
        value: value ?? 1,
        recordedAt: new Date(),
      },
    });
  }
}

export const abTest = new ABTestService();
```

---

## APIエンドポイントでの使用

```typescript
// src/routes/checkout.ts
router.get('/checkout', authenticate, async (req, res) => {
  const userId = req.user.id;

  // A/Bテスト: 新チェックアウトUIを50%のユーザーに表示
  const variant = await abTest.getVariant(userId, 'checkout_redesign');

  res.render('checkout', {
    useNewUI: variant?.variantName === 'new_checkout',
    userId,
  });
});

// コンバージョン記録
router.post('/orders', authenticate, async (req, res) => {
  const order = await createOrder(req.user.id, req.body);

  // 購入完了をA/Bテストに記録
  await abTest.trackConversion(req.user.id, 'checkout_redesign', 'purchase', order.total);

  res.status(201).json(order);
});
```

---

## 統計的有意性の計算

```typescript
// src/experiments/statistics.ts

interface VariantStats {
  variantName: string;
  visitors: number;
  conversions: number;
  conversionRate: number;
}

// z検定による統計的有意性（二項比率の差）
function zTest(control: VariantStats, variant: VariantStats): {
  pValue: number;
  significant: boolean;
  lift: number;
} {
  const p1 = control.conversions / control.visitors;
  const p2 = variant.conversions / variant.visitors;

  // プールされた比率
  const p = (control.conversions + variant.conversions) /
             (control.visitors + variant.visitors);

  const se = Math.sqrt(p * (1 - p) * (1 / control.visitors + 1 / variant.visitors));
  const z = (p2 - p1) / se;

  // p値の近似（両側検定）
  const pValue = 2 * (1 - normalCDF(Math.abs(z)));

  return {
    pValue,
    significant: pValue < 0.05,
    lift: ((p2 - p1) / p1) * 100, // コンバージョン率の改善率（%）
  };
}

// GET /api/experiments/checkout_redesign/results
router.get('/experiments/:name/results', async (req, res) => {
  const stats = await prisma.$queryRaw<VariantStats[]>`
    SELECT
      v.name as "variantName",
      COUNT(DISTINCT ea.user_id) as visitors,
      COUNT(DISTINCT ee.user_id) as conversions,
      COUNT(DISTINCT ee.user_id)::float / COUNT(DISTINCT ea.user_id) as "conversionRate"
    FROM experiment_assignments ea
    JOIN variants v ON ea.variant_id = v.id
    LEFT JOIN experiment_events ee
      ON ea.experiment_id = ee.experiment_id
      AND ea.variant_id = ee.variant_id
      AND ea.user_id = ee.user_id
      AND ee.event_name = 'purchase'
    WHERE ea.experiment_id = (
      SELECT id FROM experiments WHERE name = ${req.params.name}
    )
    GROUP BY v.name
  `;

  const control = stats.find(s => s.variantName === 'control')!;
  const variants = stats.filter(s => s.variantName !== 'control');

  const results = variants.map(variant => ({
    ...variant,
    ...zTest(control, variant),
  }));

  res.json({ control, variants: results });
});
```

---

## まとめ

Claude CodeでA/Bテストを設計する：

1. **CLAUDE.md** にコホート分割方法・実験期間・統計基準を明記
2. **SHA256ハッシュ** でユーザーIDから決定論的バリアント割り当て（ページリロードで変わらない）
3. **既存割り当てを再利用** でユーザー体験の一貫性を保証
4. **z検定** で統計的有意性を計算（感覚ではなく数値で判断）

---

*A/Bテスト設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
