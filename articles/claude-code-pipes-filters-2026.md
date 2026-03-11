---
title: "Claude CodeでPipes and Filtersパターンを設計する：データ変換パイプライン・ミドルウェアチェーン・型安全な処理"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "redis", "architecture"]
published: true
published_at: "2026-03-18 15:00"
---

## はじめに

「データ変換の処理が増えるたびにサービスが肥大化している」——Pipes and Filtersパターンで各変換ステップを独立したFilterにし、Pipeline（パイプ）で組み合わせて再利用可能なデータ変換チェーンを構築する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにPipes and Filters設計ルールを書く

```markdown
## Pipes and Filters設計ルール

### Filterの設計
- 各Filterは単一責任（1つの変換だけを行う）
- 入力と出力の型を明示（TypeScript Generics）
- Filterはステートレス（状態を持たない）
- エラーはFilterがスローしてPipelineがキャッチ

### Pipeline構成
- Filterを配列で順番に指定
- 途中でエラーが出たら後続のFilterは実行しない
- 非同期FilterとStream Filterを混在可能

### テスト容易性
- 各Filterは独立してテスト可能（入力→出力の純粋関数）
- Pipelineは結合テスト（全Filterの連結）
```

---

## Pipes and Filters実装の生成

```
Pipes and Filtersパターンを設計してください。

要件：
- 型安全なGenericパイプライン
- 非同期Filter対応
- エラーハンドリング
- 条件付きFilter（スキップ可能）

生成ファイル: src/pipeline/
```

---

## 生成されるPipes and Filters実装

```typescript
// src/pipeline/pipeline.ts — 型安全なパイプラインエンジン

export type Filter<TIn, TOut> = (input: TIn, context: PipelineContext) => Promise<TOut> | TOut;

export interface ConditionalFilter<TIn, TOut> {
  filter: Filter<TIn, TOut>;
  when?: (input: TIn, context: PipelineContext) => boolean;
}

export interface PipelineContext {
  pipelineId: string;
  startedAt: Date;
  metadata: Record<string, unknown>;
  filterResults: Map<string, unknown>;
}

// 型推論でパイプラインの入出力型を連鎖させるビルダー
export class Pipeline<T> {
  private readonly filters: Array<ConditionalFilter<unknown, unknown>> = [];

  pipe<TOut>(
    filter: Filter<T, TOut>,
    options: { when?: (input: T, ctx: PipelineContext) => boolean; name?: string } = {}
  ): Pipeline<TOut> {
    this.filters.push({
      filter: filter as Filter<unknown, unknown>,
      when: options.when as ((input: unknown, ctx: PipelineContext) => boolean) | undefined,
    });
    return this as unknown as Pipeline<TOut>;
  }

  async execute(input: T): Promise<T> {
    const context: PipelineContext = {
      pipelineId: ulid(),
      startedAt: new Date(),
      metadata: {},
      filterResults: new Map(),
    };

    let current: unknown = input;

    for (let i = 0; i < this.filters.length; i++) {
      const { filter, when } = this.filters[i];

      // 条件フィルタ: whenがfalseなら現在の値をそのまま通す
      if (when && !when(current, context)) {
        logger.debug({ pipelineId: context.pipelineId, filterIndex: i }, 'Filter skipped by condition');
        continue;
      }

      const filterStartMs = Date.now();

      try {
        const result = await filter(current, context);
        context.filterResults.set(`filter-${i}`, result);
        current = result;

        logger.debug(
          { pipelineId: context.pipelineId, filterIndex: i, durationMs: Date.now() - filterStartMs },
          'Filter completed'
        );
      } catch (error) {
        logger.error(
          { pipelineId: context.pipelineId, filterIndex: i, error },
          'Pipeline filter failed'
        );
        throw new PipelineError(`Filter ${i} failed: ${(error as Error).message}`, i, error as Error);
      }
    }

    return current as T;
  }
}

export class PipelineError extends Error {
  constructor(message: string, public readonly filterIndex: number, public readonly cause: Error) {
    super(message);
  }
}
```

```typescript
// src/pipeline/filters/orderFilters.ts — 注文処理パイプラインのFilter群

interface RawOrderInput {
  userId: string;
  items: Array<{ productId: string; quantity: number }>;
  couponCode?: string;
}

interface ValidatedOrder extends RawOrderInput {
  items: Array<{ productId: string; quantity: number; price: number; stockAvailable: boolean }>;
  user: User;
}

interface PricedOrder extends ValidatedOrder {
  subtotal: number;
  discountAmount: number;
  totalAmount: number;
}

interface TaxedOrder extends PricedOrder {
  taxAmount: number;
  finalAmount: number;
}

// Filter 1: バリデーション + データエンリッチメント
const validateAndEnrichFilter: Filter<RawOrderInput, ValidatedOrder> = async (input) => {
  const user = await userService.getUser(input.userId);
  if (!user) throw new ValidationError(`User ${input.userId} not found`);

  const enrichedItems = await Promise.all(
    input.items.map(async item => {
      const product = await productService.getProduct(item.productId);
      const stock = await inventoryService.checkStock(item.productId, item.quantity);
      return { ...item, price: product.price, stockAvailable: stock.available };
    })
  );

  const outOfStock = enrichedItems.filter(i => !i.stockAvailable);
  if (outOfStock.length > 0) {
    throw new OutOfStockError(outOfStock.map(i => i.productId));
  }

  return { ...input, items: enrichedItems, user };
};

// Filter 2: 価格計算 + クーポン適用
const calculatePriceFilter: Filter<ValidatedOrder, PricedOrder> = async (input) => {
  const subtotal = input.items.reduce((sum, item) => sum + item.price * item.quantity, 0);

  let discountAmount = 0;
  if (input.couponCode) {
    const coupon = await couponService.validate(input.couponCode, subtotal);
    discountAmount = coupon.discountAmount;
  }

  return { ...input, subtotal, discountAmount, totalAmount: subtotal - discountAmount };
};

// Filter 3: 税計算（任意: 課税ユーザーのみ）
const calculateTaxFilter: Filter<PricedOrder, TaxedOrder> = async (input) => {
  const taxRate = await taxService.getTaxRate(input.user.country);
  const taxAmount = Math.round(input.totalAmount * taxRate);
  return { ...input, taxAmount, finalAmount: input.totalAmount + taxAmount };
};

// Filter 4: 詐欺チェック
const fraudCheckFilter: Filter<TaxedOrder, TaxedOrder> = async (input) => {
  const riskScore = await fraudService.calculateRisk(input.userId, input.finalAmount);
  if (riskScore > 0.8) throw new FraudDetectedError(`High risk score: ${riskScore}`);
  return input;
};

// パイプライン組み立て
export function createOrderPipeline() {
  return new Pipeline<RawOrderInput>()
    .pipe(validateAndEnrichFilter)
    .pipe(calculatePriceFilter)
    .pipe(calculateTaxFilter, {
      // 課税対象ユーザーのみ税計算
      when: (input, _ctx) => (input as ValidatedOrder).user.taxable === true,
    })
    .pipe(fraudCheckFilter, {
      // 金額が1万円以上の場合のみ詐欺チェック
      when: (input, _ctx) => (input as PricedOrder).totalAmount >= 10_000,
    });
}

// 使用例
const pipeline = createOrderPipeline();

const processedOrder = await pipeline.execute({
  userId: 'user-123',
  items: [{ productId: 'prod-abc', quantity: 2 }],
  couponCode: 'SAVE10',
});
```

---

## まとめ

Claude CodeでPipes and Filtersパターンを設計する：

1. **CLAUDE.md** に各Filterは単一責任・ステートレス・TypeScript Genericsで入出力型を明示・エラーはFilterがスローしてPipelineがインデックスとともに記録を明記
2. **型推論による連鎖** `Pipeline<RawInput>.pipe(filter1).pipe(filter2)`でTypeScriptが各Filterの入出力型を自動推論——Filter2はFilter1の出力型しか受け取れない型安全なチェーン
3. **条件付きFilter（`when`オプション）** でユーザー属性や金額に応じてFilterをスキップ——「課税ユーザーにだけ税計算、1万円以上にだけ詐欺チェック」を宣言的に記述
4. **FilterResultsのコンテキスト保存** で後続Filterが前のFilterの結果を参照可能——「Filter2が計算した小計をFilter4が参照する」といった依存関係をContextで解決

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
