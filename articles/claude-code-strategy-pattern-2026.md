---
title: "Claude Codeでストラテジーパターンを設計する：アルゴリズムの差し替え・設定ドリブンな戦略選択・テスト容易性"
emoji: "♟️"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "architecture"]
published: true
published_at: "2026-03-21 16:00"
---

## はじめに

「決済方法ごとにif-elseが増えてコードが膨らむ」「新しい送料計算ロジックを追加するたびに既存コードを変更している」——ストラテジーパターンでアルゴリズムを差し替え可能にし、Open/Closed原則を実現する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにストラテジーパターン設計ルールを書く

```markdown
## ストラテジーパターン設計ルール

### 適用場面
- 同じ目的で複数のアルゴリズムが存在する（送料計算、割引計算、通知送信）
- アルゴリズムが設定・ユーザータイプ・リクエストで動的に切り替わる
- 新しいアルゴリズムを追加しても既存コードを変更したくない

### 設計
- Strategy Interface: 共通インターフェース（execute/calculate等）
- Concrete Strategy: 各アルゴリズムの実装
- Context: ストラテジーを保持・呼び出す

### TypeScriptでの実装
- インターフェースで型定義
- DIコンテナまたはファクトリーでストラテジーを注入
- 設定ファイル/DBでストラテジーを動的選択
```

---

## ストラテジーパターン実装の生成

```
ストラテジーパターンを設計してください。

要件：
- 送料計算の複数ストラテジー
- 設定ドリブンな動的選択
- A/Bテスト対応
- テスト可能な設計

生成ファイル: src/domain/shipping/
```

---

## 生成されるストラテジーパターン実装

```typescript
// src/domain/shipping/shippingStrategy.ts — ストラテジーインターフェース

export interface ShippingInput {
  weight: number;           // g
  distance: number;         // km
  isPremiumUser: boolean;
  itemCount: number;
  destination: {
    prefecture: string;
    isRemote: boolean;      // 離島・山間部
  };
}

export interface ShippingResult {
  fee: Money;
  estimatedDays: number;
  carrier: string;
  label: string;
}

// ストラテジーインターフェース
export interface IShippingStrategy {
  readonly strategyId: string;
  calculate(input: ShippingInput): ShippingResult;
  isApplicable(input: ShippingInput): boolean;  // このストラテジーが適用可能か
}
```

```typescript
// src/domain/shipping/strategies/ — 具体的ストラテジー群

// 1. 標準配送
export class StandardShippingStrategy implements IShippingStrategy {
  readonly strategyId = 'standard';

  calculate(input: ShippingInput): ShippingResult {
    let baseFee = 500;
    if (input.weight > 2000) baseFee += Math.ceil((input.weight - 2000) / 1000) * 100;
    if (input.destination.isRemote) baseFee += 300;
    if (input.isPremiumUser) baseFee = Math.max(0, baseFee - 200);

    return {
      fee: Money.of(baseFee, 'JPY'),
      estimatedDays: input.destination.isRemote ? 4 : 3,
      carrier: 'Yamato',
      label: '標準配送',
    };
  }

  isApplicable(_input: ShippingInput): boolean { return true; }  // 常に適用可能
}

// 2. 速達配送
export class ExpressShippingStrategy implements IShippingStrategy {
  readonly strategyId = 'express';

  calculate(input: ShippingInput): ShippingResult {
    const baseFee = input.isPremiumUser ? 800 : 1200;
    return {
      fee: Money.of(baseFee, 'JPY'),
      estimatedDays: 1,
      carrier: 'Sagawa Express',
      label: '速達（翌日配送）',
    };
  }

  isApplicable(input: ShippingInput): boolean {
    return !input.destination.isRemote && input.weight <= 5000;
  }
}

// 3. まとめ配送（複数商品の場合に安くなる）
export class BulkShippingStrategy implements IShippingStrategy {
  readonly strategyId = 'bulk';

  calculate(input: ShippingInput): ShippingResult {
    // アイテム数が多いほど単価が下がる
    const feePerItem = input.itemCount >= 10 ? 50 : input.itemCount >= 5 ? 80 : 100;
    const totalFee = feePerItem * input.itemCount;

    return {
      fee: Money.of(totalFee, 'JPY'),
      estimatedDays: 5,
      carrier: 'Fukuyama Transporting',
      label: 'まとめ便（お得）',
    };
  }

  isApplicable(input: ShippingInput): boolean {
    return input.itemCount >= 3;
  }
}

// 4. 無料配送（プレミアム会員向け）
export class FreeShippingStrategy implements IShippingStrategy {
  readonly strategyId = 'free';

  calculate(input: ShippingInput): ShippingResult {
    return {
      fee: Money.zero('JPY'),
      estimatedDays: 3,
      carrier: 'Yamato',
      label: '送料無料（プレミアム特典）',
    };
  }

  isApplicable(input: ShippingInput): boolean {
    return input.isPremiumUser && !input.destination.isRemote;
  }
}
```

```typescript
// src/domain/shipping/shippingCalculator.ts — コンテキスト（ストラテジーを管理）

export class ShippingCalculator {
  private readonly strategies: IShippingStrategy[];

  constructor(strategies: IShippingStrategy[]) {
    // 優先度順で登録（最初に適用可能なものが選ばれる）
    this.strategies = strategies;
  }

  // 最適なストラテジーを選択して計算
  calculate(input: ShippingInput): ShippingResult {
    const applicableStrategies = this.strategies.filter(s => s.isApplicable(input));

    if (applicableStrategies.length === 0) {
      throw new Error('No applicable shipping strategy found');
    }

    // 最初に見つかった（優先度最高）ストラテジーを使用
    return applicableStrategies[0].calculate(input);
  }

  // 全ての適用可能なストラテジーの候補を返す（UI表示用）
  getOptions(input: ShippingInput): Array<ShippingResult & { strategyId: string }> {
    return this.strategies
      .filter(s => s.isApplicable(input))
      .map(s => ({ ...s.calculate(input), strategyId: s.strategyId }));
  }

  // ストラテジーを名前で指定して実行（ユーザーが選択した場合）
  calculateWith(strategyId: string, input: ShippingInput): ShippingResult {
    const strategy = this.strategies.find(s => s.strategyId === strategyId);
    if (!strategy) throw new StrategyNotFoundError(strategyId);
    if (!strategy.isApplicable(input)) throw new StrategyNotApplicableError(strategyId);
    return strategy.calculate(input);
  }
}

// 設定ドリブンな動的選択（A/Bテスト対応）
export class ConfigurableShippingCalculator {
  constructor(
    private readonly baseCalculator: ShippingCalculator,
    private readonly featureFlags: IFeatureFlags
  ) {}

  async calculate(input: ShippingInput, userId: string): Promise<ShippingResult> {
    // A/Bテスト: 50%のユーザーに新しい送料ロジックを適用
    const useNewPricing = await this.featureFlags.isEnabled('new-shipping-pricing', userId);

    if (useNewPricing) {
      return this.calculateWithNewPricing(input);
    }

    return this.baseCalculator.calculate(input);
  }

  private calculateWithNewPricing(input: ShippingInput): ShippingResult {
    // 新しい計算ロジック（A/Bテスト中）
    return {
      fee: Money.of(300, 'JPY'),  // フラット料金
      estimatedDays: 2,
      carrier: 'JPost',
      label: '新料金プラン（β）',
    };
  }
}

// DIコンテナでの設定
const shippingCalculator = new ShippingCalculator([
  new FreeShippingStrategy(),    // 最優先: プレミアム無料配送
  new ExpressShippingStrategy(), // 次: 速達
  new BulkShippingStrategy(),    // 次: まとめ便
  new StandardShippingStrategy(), // 最後: 標準配送
]);

// 使用例
const options = shippingCalculator.getOptions({
  weight: 500,
  distance: 200,
  isPremiumUser: true,
  itemCount: 3,
  destination: { prefecture: '東京都', isRemote: false },
});
// → [
//   { strategyId: 'free', fee: ¥0, label: '送料無料（プレミアム特典）' },
//   { strategyId: 'express', fee: ¥800, label: '速達（翌日配送）' },
//   { strategyId: 'bulk', fee: ¥240, label: 'まとめ便（お得）' },
//   { strategyId: 'standard', fee: ¥300, label: '標準配送' },
// ]
```

---

## まとめ

Claude Codeでストラテジーパターンを設計する：

1. **CLAUDE.md** にIShippingStrategyインターフェースで全ストラテジーを同一形式に・コンテキスト（ShippingCalculator）がif-elseなしにストラテジーを選択・新ストラテジー追加は既存コードに手を加えない（Open/Closed）を明記
2. **`isApplicable()`** でストラテジーの適用可否を各クラスが自己判断——`FreeShippingStrategy.isApplicable(input)`が`isPremiumUser && !isRemote`を判定。コンテキストはif-elseを書かずに適用可能なストラテジーをフィルタリングできる
3. **`getOptions()`** でUIに全候補を一覧提供——「送料無料・速達・まとめ便・標準」の全オプションと料金をまとめて返す。ユーザーが選択したら`calculateWith(strategyId)`で実行
4. **A/Bテスト対応** ——`ConfigurableShippingCalculator`がFeature Flagと組み合わせて50%のユーザーに新料金ロジックを適用。新ストラテジーをコードに追加するだけでFeature Flagで段階的に展開できる

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
