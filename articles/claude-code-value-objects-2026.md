---
title: "Claude Codeで値オブジェクトを設計する：不変性・等値性・ドメイン表現の型安全化"
emoji: "💎"
type: "tech"
topics: ["claudecode", "nodejs", "typescript", "postgresql", "architecture"]
published: true
published_at: "2026-03-18 21:00"
---

## はじめに

「金額を`number`で扱っていて通貨単位の混在バグが起きた」「メールアドレスを`string`で渡し続けていてバリデーションが散らばった」——値オブジェクトでドメイン概念を型として表現し、不正な値が作れない設計をClaude Codeに生成させる。

---

## CLAUDE.mdに値オブジェクト設計ルールを書く

```markdown
## 値オブジェクト設計ルール

### 特性
- 不変（Immutable）: 作成後に変更不可（全フィールドreadonly）
- 等値性: IDではなく値で等しいかを判定（equals()メソッド）
- 自己検証: コンストラクタでバリデーション（無効な値は作れない）

### 設計
- プリミティブ型（string/number）の代わりに使用
- privateコンストラクタ + static create()でファクトリーメソッド
- toValue()でプリミティブに変換（DBへの保存など）

### 適用対象
- 金額・通貨・価格
- メールアドレス・URLなどの文字列
- 住所・座標などの複合値
- 数量・サイズ（単位付き数値）
```

---

## 値オブジェクト実装の生成

```
値オブジェクトを設計してください。

要件：
- 不変・等値性・自己検証
- Moneyクラス（金額+通貨）
- EmailAddress
- URL
- Quantity（単位付き数値）

生成ファイル: src/domain/valueObjects/
```

---

## 生成される値オブジェクト実装

```typescript
// src/domain/valueObjects/money.ts — 金額値オブジェクト

export type Currency = 'JPY' | 'USD' | 'EUR' | 'GBP';

export class Money {
  private constructor(
    private readonly _amount: number,  // 最小単位（円/セント）
    private readonly _currency: Currency
  ) {}

  static of(amount: number, currency: Currency): Money {
    if (!Number.isInteger(amount)) {
      throw new DomainError(`Money amount must be an integer (got ${amount})`);
    }
    if (amount < 0) {
      throw new DomainError(`Money amount cannot be negative (got ${amount})`);
    }
    return new Money(amount, currency);
  }

  static zero(currency: Currency): Money {
    return new Money(0, currency);
  }

  add(other: Money): Money {
    if (this._currency !== other._currency) {
      throw new CurrencyMismatchError(this._currency, other._currency);
    }
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other: Money): Money {
    if (this._currency !== other._currency) {
      throw new CurrencyMismatchError(this._currency, other._currency);
    }
    if (this._amount < other._amount) {
      throw new DomainError('Insufficient amount');
    }
    return new Money(this._amount - other._amount, this._currency);
  }

  multiply(factor: number): Money {
    if (factor < 0) throw new DomainError('Factor cannot be negative');
    return new Money(Math.round(this._amount * factor), this._currency);
  }

  isGreaterThan(other: Money): boolean {
    this.ensureSameCurrency(other);
    return this._amount > other._amount;
  }

  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  toValue(): { amount: number; currency: Currency } {
    return { amount: this._amount, currency: this._currency };
  }

  toString(): string {
    const formatter = new Intl.NumberFormat('ja-JP', {
      style: 'currency',
      currency: this._currency,
    });
    return formatter.format(this._currency === 'JPY' ? this._amount : this._amount / 100);
  }

  get amount(): number { return this._amount; }
  get currency(): Currency { return this._currency; }

  private ensureSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new CurrencyMismatchError(this._currency, other._currency);
    }
  }
}

// src/domain/valueObjects/emailAddress.ts — メールアドレス値オブジェクト

export class EmailAddress {
  private readonly _value: string;

  private constructor(value: string) {
    this._value = value.toLowerCase().trim();
  }

  static create(email: string): EmailAddress {
    const normalized = email.toLowerCase().trim();
    const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

    if (!EMAIL_REGEX.test(normalized)) {
      throw new DomainError(`Invalid email address: ${email}`);
    }
    if (normalized.length > 255) {
      throw new DomainError('Email address too long');
    }

    return new EmailAddress(normalized);
  }

  equals(other: EmailAddress): boolean {
    return this._value === other._value;
  }

  get domain(): string {
    return this._value.split('@')[1];
  }

  toValue(): string {
    return this._value;
  }

  toString(): string {
    return this._value;
  }
}

// src/domain/valueObjects/quantity.ts — 数量値オブジェクト（単位付き）

export type QuantityUnit = 'piece' | 'kg' | 'liter' | 'meter';

export class Quantity {
  private constructor(
    private readonly _value: number,
    private readonly _unit: QuantityUnit
  ) {}

  static of(value: number, unit: QuantityUnit): Quantity {
    if (value < 0) throw new DomainError('Quantity cannot be negative');
    if (!Number.isFinite(value)) throw new DomainError('Quantity must be finite');
    return new Quantity(value, unit);
  }

  add(other: Quantity): Quantity {
    if (this._unit !== other._unit) {
      throw new UnitMismatchError(this._unit, other._unit);
    }
    return new Quantity(this._value + other._value, this._unit);
  }

  isEnough(required: Quantity): boolean {
    if (this._unit !== required._unit) throw new UnitMismatchError(this._unit, required._unit);
    return this._value >= required._value;
  }

  equals(other: Quantity): boolean {
    return this._value === other._value && this._unit === other._unit;
  }

  toValue(): { value: number; unit: QuantityUnit } {
    return { value: this._value, unit: this._unit };
  }

  get value(): number { return this._value; }
  get unit(): QuantityUnit { return this._unit; }
}
```

```typescript
// 使用例: 値オブジェクトを使ったドメインモデル

class OrderItem {
  constructor(
    readonly productId: string,
    readonly price: Money,      // numberではなくMoney
    readonly quantity: Quantity // numberではなくQuantity
  ) {}

  get subtotal(): Money {
    return this.price.multiply(this.quantity.value);
  }
}

class Order {
  constructor(
    readonly id: string,
    readonly customerEmail: EmailAddress,  // stringではなくEmailAddress
    readonly items: OrderItem[]
  ) {}

  get total(): Money {
    if (this.items.length === 0) return Money.zero('JPY');
    return this.items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero(this.items[0].price.currency)
    );
  }
}

// 型安全な使用例
const price = Money.of(1980, 'JPY');
const qty = Quantity.of(3, 'piece');
const email = EmailAddress.create('customer@example.com');

const item = new OrderItem('prod-1', price, qty);
console.log(item.subtotal.toString()); // "¥5,940"

// 通貨混在を型でブロック
const usdPrice = Money.of(100, 'USD');
try {
  price.add(usdPrice); // CurrencyMismatchError: JPY vs USD
} catch (e) {
  console.error(e.message);
}

// Prismaでの保存/復元
const savedOrder = await prisma.order.create({
  data: {
    customerEmail: email.toValue(),  // string
    totalAmount: total.amount,       // number
    currency: total.currency,        // string
  },
});

// 復元
const restoredEmail = EmailAddress.create(savedOrder.customerEmail);
const restoredTotal = Money.of(savedOrder.totalAmount, savedOrder.currency as Currency);
```

---

## まとめ

Claude Codeで値オブジェクトを設計する：

1. **CLAUDE.md** にprivateコンストラクタ + static create()・不変（readonlyフィールド）・equals()で等値比較・toValue()でプリミティブ変換を明記
2. **通貨混在防止（Money）** で`price.add(usdPrice)` → `CurrencyMismatchError`——TypeScriptの型では`Money`同士なら防げないが、実行時に通貨を検証することで「JPY金額にUSD金額を加算」バグを確実に防止
3. **自己検証コンストラクタ** でEmailAddressの不正な値が作れない——`EmailAddress.create('invalid')`でDomainErrorを即スロー。バリデーションが値オブジェクトに集約されてサービス層に散らばらない
4. **`toValue()`でプリミティブ変換** してDBへの保存と値オブジェクトの使用を分離——Prismaには`email.toValue()`でstringを渡し、取得後は`EmailAddress.create(row.email)`で復元

---

*アーキテクチャ設計のレビューは **Code Review Pack（¥980）** の `/code-review` で確認できます。*

*[prompt-works.jp](https://prompt-works.jp)*

*みょうが (@myougaTheAxo) — ウーパールーパーのVTuber。*
