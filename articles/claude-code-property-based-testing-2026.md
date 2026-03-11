---
title: "Claude Codeでプロパティベーステストを設計する：fast-check・生成戦略・不変条件の発見"
emoji: "🎲"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "testing", "architecture"]
published: true
published_at: "2026-03-20 14:00"
---

## はじめに

「手書きのテストケースで思いつかない境界値があった」「数値計算のバグが特定の入力でのみ発生する」——プロパティベーステスト（PBT）でランダム生成した大量の入力に対して不変条件を検証し、手書きでは見つけられないバグを発見する設計をClaude Codeに生成させる。

---

## CLAUDE.mdにプロパティベーステスト設計ルールを書く

```markdown
## プロパティベーステスト設計ルール

### 適用場面
- 数値計算・金額計算（精度・オーバーフロー）
- コレクション操作（ソート・マージ・フィルタ）
- シリアライズ/デシリアライズの往復
- ドメインルール（不変条件）の検証

### プロパティの書き方
- 単一テストケースではなく「任意の入力Xに対してYが成立する」を記述
- 例: 「任意の金額を加算して減算すると元の金額に戻る」
- 例: 「ソート後の配列は長さが変わらず要素が昇順になる」

### ライブラリ
- fast-check: TypeScript向けPBTライブラリ（推奨）
- shrinkage: 失敗時に最小再現ケースに自動縮小
```

---

## プロパティベーステスト実装の生成

```
プロパティベーステストを設計してください。

要件：
- fast-checkによる生成戦略
- ドメインオブジェクトのカスタムArbitrary
- 不変条件テスト
- シュリンキング活用

生成ファイル: src/__tests__/properties/
```

---

## 生成されるプロパティベーステスト実装

```typescript
// src/__tests__/properties/money.property.test.ts — Moneyクラスのプロパティテスト

import fc from 'fast-check';

// カスタムArbitrary: 有効なMoneyオブジェクトを生成
const validAmount = () =>
  fc.integer({ min: 0, max: 10_000_000 });  // 0円〜1000万円

const validCurrency = () =>
  fc.constantFrom('JPY', 'USD', 'EUR');

const validMoney = () =>
  fc.record({
    amount: validAmount(),
    currency: validCurrency(),
  }).map(({ amount, currency }) => Money.of(amount, currency));

const sameCurrencyMoneyPair = () =>
  fc.record({
    currency: validCurrency(),
    amount1: validAmount(),
    amount2: validAmount(),
  }).map(({ currency, amount1, amount2 }) => ({
    a: Money.of(amount1, currency),
    b: Money.of(amount2, currency),
  }));

describe('Money プロパティテスト', () => {
  // 加算の交換法則: a + b = b + a
  it('加算は交換法則を満たす', () => {
    fc.assert(
      fc.property(sameCurrencyMoneyPair(), ({ a, b }) => {
        expect(a.add(b).equals(b.add(a))).toBe(true);
      }),
      { numRuns: 1000 }
    );
  });

  // 加算の結合法則: (a + b) + c = a + (b + c)
  it('加算は結合法則を満たす', () => {
    fc.assert(
      fc.property(
        validCurrency().chain(currency =>
          fc.tuple(
            fc.integer({ min: 0, max: 1_000_000 }),
            fc.integer({ min: 0, max: 1_000_000 }),
            fc.integer({ min: 0, max: 1_000_000 }),
          ).map(([a, b, c]) => ({
            a: Money.of(a, currency),
            b: Money.of(b, currency),
            c: Money.of(c, currency),
          }))
        ),
        ({ a, b, c }) => {
          const left = a.add(b).add(c);
          const right = a.add(b.add(c));
          expect(left.equals(right)).toBe(true);
        }
      ),
      { numRuns: 1000 }
    );
  });

  // 加算後の減算で元に戻る（逆演算）
  it('加算→減算で元の金額に戻る', () => {
    fc.assert(
      fc.property(sameCurrencyMoneyPair(), ({ a, b }) => {
        expect(a.add(b).subtract(b).equals(a)).toBe(true);
      }),
      { numRuns: 1000 }
    );
  });

  // ゼロ元: a + 0 = a
  it('ゼロを加算しても変わらない', () => {
    fc.assert(
      fc.property(validMoney(), (money) => {
        const zero = Money.zero(money.currency);
        expect(money.add(zero).equals(money)).toBe(true);
      }),
      { numRuns: 500 }
    );
  });

  // 通貨不一致でエラー
  it('異なる通貨の加算はエラーになる', () => {
    fc.assert(
      fc.property(
        fc.tuple(
          fc.constantFrom('JPY', 'USD', 'EUR'),
          fc.constantFrom('JPY', 'USD', 'EUR'),
        ).filter(([c1, c2]) => c1 !== c2),
        fc.integer({ min: 1, max: 1000 }),
        ([currency1, currency2], amount) => {
          const a = Money.of(amount, currency1);
          const b = Money.of(amount, currency2);
          expect(() => a.add(b)).toThrow(CurrencyMismatchError);
        }
      ),
      { numRuns: 200 }
    );
  });
});
```

```typescript
// src/__tests__/properties/order.property.test.ts — Order集約のプロパティテスト

// カスタムArbitrary: 有効な注文アイテムを生成
const validOrderItem = () =>
  fc.record({
    productId: fc.string({ minLength: 1, maxLength: 50 }).map(s => `prod-${s}`),
    quantity: fc.integer({ min: 1, max: 100 }),
    price: fc.integer({ min: 1, max: 100_000 }),
  }).map(({ productId, quantity, price }) => ({
    productId,
    quantity,
    price: Money.of(price, 'JPY'),
  }));

// 1〜10アイテムのリストを生成（重複productIdなし）
const validItemList = () =>
  fc.array(validOrderItem(), { minLength: 1, maxLength: 10 })
    .map(items => {
      // productIdの重複を除去
      const seen = new Set<string>();
      return items.filter(item => {
        if (seen.has(item.productId)) return false;
        seen.add(item.productId);
        return true;
      });
    })
    .filter(items => items.length > 0);

describe('Order 集約プロパティテスト', () => {
  // 注文の合計 = 各アイテムの小計の合計
  it('注文合計はアイテムの小計の和と一致する', () => {
    fc.assert(
      fc.property(fc.uuid(), validItemList(), (userId, items) => {
        const order = Order.create(userId, items);
        const expectedTotal = items.reduce(
          (sum, item) => sum + item.quantity * item.price.amount,
          0
        );
        expect(order.total.amount).toBe(expectedTotal);
      }),
      { numRuns: 500 }
    );
  });

  // アイテム追加後のアイテム数
  it('アイテム追加後は必ずアイテム数が増加する（または既存数量が増える）', () => {
    fc.assert(
      fc.property(
        fc.uuid(),
        validItemList(),
        fc.record({
          productId: fc.string({ minLength: 5 }).map(s => `new-${s}`),
          quantity: fc.integer({ min: 1, max: 10 }),
          price: fc.integer({ min: 100, max: 10_000 }),
        }),
        (userId, items, newItem) => {
          const order = Order.create(userId, items);
          const beforeCount = order.items.length;
          const beforeTotal = order.total.amount;

          order.addItem(
            `new-${newItem.productId}`,
            newItem.quantity,
            Money.of(newItem.price, 'JPY')
          );

          expect(order.items.length).toBeGreaterThan(beforeCount);
          expect(order.total.amount).toBeGreaterThan(beforeTotal);
        }
      ),
      { numRuns: 300 }
    );
  });

  // シリアライズ/デシリアライズの往復一致
  it('DBへの保存→復元で注文データが一致する', () => {
    fc.assert(
      fc.property(fc.uuid(), validItemList(), (userId, items) => {
        const original = Order.create(userId, items);

        // Mapper経由でDB形式に変換して復元
        const dbData = OrderMapper.toCreateData(original);
        const restored = OrderMapper.toDomain({
          ...dbData,
          items: original.items.map((item, i) => ({
            id: `item-${i}`,
            orderId: original.id,
            productId: item.productId,
            quantity: item.quantity,
            price: item.subtotal.amount / item.quantity,
            currency: 'JPY',
          })),
          createdAt: original.createdAt,
        });

        expect(restored.id).toBe(original.id);
        expect(restored.userId).toBe(original.userId);
        expect(restored.total.amount).toBe(original.total.amount);
        expect(restored.items.length).toBe(original.items.length);
      }),
      { numRuns: 200 }
    );
  });
});
```

```typescript
// src/__tests__/properties/pagination.property.test.ts — ページネーションのプロパティテスト

describe('カーソルページネーション プロパティテスト', () => {
  const paginator = new CursorPaginator();

  // 全ページを走査すると重複なく全アイテムを取得できる
  it('全ページ走査で重複なく全件取得できる', async () => {
    await fc.assert(
      fc.asyncProperty(
        fc.array(fc.record({
          id: fc.uuid(),
          value: fc.integer({ min: 0, max: 1000 }),
        }), { minLength: 0, maxLength: 100 }),
        fc.integer({ min: 1, max: 20 }),  // ページサイズ
        async (allItems, pageSize) => {
          // インメモリDBに格納
          const db = new InMemoryItemDb(allItems);

          const collected: typeof allItems = [];
          let cursor: string | undefined;

          while (true) {
            const page = await paginator.paginate(db, { cursor, limit: pageSize });
            collected.push(...page.items);

            if (!page.hasMore) break;
            cursor = page.nextCursor;

            // 無限ループ防止
            if (collected.length > allItems.length + 10) break;
          }

          // 全件取得できている
          expect(collected.length).toBe(allItems.length);

          // IDに重複がない
          const ids = new Set(collected.map(i => i.id));
          expect(ids.size).toBe(collected.length);
        }
      ),
      { numRuns: 100 }
    );
  });

  // カーソルエンコード/デコードの往復
  it('カーソルのエンコード→デコードは情報を失わない', () => {
    fc.assert(
      fc.property(
        fc.record({
          createdAt: fc.date({ min: new Date('2020-01-01'), max: new Date('2030-01-01') }),
          id: fc.uuid(),
        }),
        ({ createdAt, id }) => {
          const encoded = CursorEncoder.encode({ createdAt, id });
          const decoded = CursorEncoder.decode(encoded);

          expect(decoded.id).toBe(id);
          expect(decoded.createdAt.getTime()).toBe(createdAt.getTime());
        }
      ),
      { numRuns: 1000 }
    );
  });
});
```

---

## まとめ

Claude Codeでプロパティベーステストを設計する：

1. **CLAUDE.md** にfast-checkで「任意の入力に対して不変条件が成立する」を記述・数値計算・コレクション操作・シリアライズ往復を優先的にPBT化を明記
2. **交換法則・結合法則・逆演算** などの数学的性質でMoneyの計算バグを網羅——手書きで`Money.of(100, 'JPY').add(Money.of(200, 'JPY'))`を書くよりも、1000回のランダム入力で「加算→減算=元値」をチェックする方がずっと信頼性が高い
3. **カスタムArbitrary** でドメインオブジェクトを直接生成——`validMoney()`・`validItemList()`を定義すれば`fc.property(validItemList(), items => ...)`と書けてテストが読みやすい
4. **シリアライズ往復テスト** でMapper/ORMのバグを発見——「DBに保存して復元した後も同じ値になる」をランダム入力で200回検証。Date型のタイムゾーン問題やJSONの精度問題がシュリンキングで最小再現ケースに縮小されて発見できる

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
