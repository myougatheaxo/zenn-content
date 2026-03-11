---
title: "Claude CodeでA/Bテストインフラを設計する：統計的有意性・実験管理・Redisトラッキング"
emoji: "🧫"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "analytics"]
published: true
published_at: "2026-03-16 22:00"
---

## はじめに

「感覚でUIを変えているが効果が測れない」——ユーザー割り当て・イベントトラッキング・統計的有意性検定まで含む軽量A/Bテストインフラを設計させる。

---

## CLAUDE.mdにA/Bテスト設計ルールを書く

```markdown
## A/Bテストインフラ設計ルール

### 実験設計
- ユーザーIDのハッシュで決定的かつ均一に割り当て
- trafficFractionで実験参加率（0-1）を設定（10%→50%→100%）

### 統計的判断
- Z検定（2プロポーション）で有意差を判定（p < 0.05）
- 最小サンプルサイズ: バリアントごとに1,000件以上
- 実験期間: 最低7日（週次サイクルの影響を排除）
```

---

## 生成されるA/Bテストインフラ実装（抜粋）

```typescript
// 決定的ユーザー割り当て
assign(experiment, userId): Variant | null {
  const participationHash = hash(`${experiment.id}:participation:${userId}`) % 100;
  if (participationHash >= experiment.trafficFraction * 100) return null;

  const variantHash = hash(`${experiment.id}:variant:${userId}`) % 100;
  let cumulative = 0;
  for (const variant of experiment.variants) {
    cumulative += variant.weight;
    if (variantHash < cumulative) return variant;
  }
}

// HyperLogLogでユニークユーザー数を近似記録
async recordConversion(experimentId, variantId, userId, metricName) {
  await redis.pfadd(`exp:${experimentId}:variant:${variantId}:metric:${metricName}:converted_users`, userId);
  await redis.incr(`exp:${experimentId}:variant:${variantId}:metric:${metricName}:conversions`);
}

// Z検定（統計的有意性）
runZTest(controlUsers, controlConversions, treatmentUsers, treatmentConversions) {
  const pooledRate = (controlConversions + treatmentConversions) / (controlUsers + treatmentUsers);
  const se = Math.sqrt(pooledRate * (1 - pooledRate) * (1/controlUsers + 1/treatmentUsers));
  const z = (treatmentRate - controlRate) / se;
  const pValue = 2 * (1 - normalCDF(Math.abs(z)));

  return {
    isSignificant: pValue < 0.05,
    relativeLift: ((treatmentRate - controlRate) / controlRate) * 100,
    recommendation: !sampleSizeAdequate ? 'wait' : !isSignificant ? 'wait' : relativeLift < 0 ? 'stop' : 'ship',
  };
}
```

---

## まとめ

1. **CLAUDE.md** にハッシュで決定的割り当て・Z検定p<0.05基準・最小1,000サンプル・実験期間7日以上を明記
2. **決定的割り当て** でリロードしても同じバリアント——セッションやCookieに依存しない
3. **HyperLogLog** でユニークユーザー数をメモリ効率良く集計
4. **recommendation（ship/wait/stop）** を自動判定——データで意思決定する文化を支援

---

*分析設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
