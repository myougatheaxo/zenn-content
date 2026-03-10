---
title: "TDDをClaude Codeで加速する：/test-genスキルでテストコードを自動生成"
emoji: "🧪"
type: "tech"
topics: ["ClaudeCode", "TDD", "Python", "JavaScript", "Testing"]
published: true
---

## TDDとは何か、なぜAIと相性がいいのか

TDD（テスト駆動開発）は「Red → Green → Refactor」のサイクルで開発を進める手法です。

1. **Red**: 失敗するテストを先に書く
2. **Green**: テストが通る最小限のコードを書く
3. **Refactor**: コードをきれいにする

みょうが的に言うと、「餌が出てくるか確認してから餌入れを設置する」みたいなものです。テスト（餌の確認）が先で、実装（餌入れ）は後。

TDDの最大の障壁は「テストを書くのが面倒」という心理的コストです。ここにClaude Codeが刺さります。

## /test-genスキルの基本的な使い方

関数を書いたら、すぐに `/test-gen` に投げます。

```bash
claude /test-gen src/utils/calculator.py
```

または、関数の実装前にテストを生成させる（純粋なTDD）：

```bash
claude "以下の関数のテストを先に書いてください。
関数: calculate_discount(price: float, coupon_code: str) -> float
仕様: クーポンコード'SAVE10'で10%割引、'HALF'で50%割引、無効なコードは0%割引"
```

## Pytest（Python）での実例

### Step 1: Red（失敗するテストを書く）

```python
# test_calculator.py
import pytest
from calculator import calculate_discount

class TestCalculateDiscount:
    def test_valid_coupon_save10(self):
        assert calculate_discount(1000.0, "SAVE10") == 900.0

    def test_valid_coupon_half(self):
        assert calculate_discount(1000.0, "HALF") == 500.0

    def test_invalid_coupon(self):
        assert calculate_discount(1000.0, "INVALID") == 1000.0

    def test_empty_coupon(self):
        assert calculate_discount(1000.0, "") == 1000.0

    def test_zero_price(self):
        assert calculate_discount(0.0, "SAVE10") == 0.0

    def test_negative_price(self):
        with pytest.raises(ValueError):
            calculate_discount(-100.0, "SAVE10")
```

まだ `calculator.py` は存在しないので、このテストは失敗します（Red）。

### Step 2: Green（テストを通す実装）

```python
# calculator.py
COUPONS = {
    "SAVE10": 0.10,
    "HALF": 0.50,
}

def calculate_discount(price: float, coupon_code: str) -> float:
    if price < 0:
        raise ValueError("Price cannot be negative")
    discount_rate = COUPONS.get(coupon_code, 0.0)
    return price * (1 - discount_rate)
```

```bash
pytest test_calculator.py -v
# 全テスト PASSED (Green)
```

### Step 3: Refactor

```python
# リファクタ後: 定数をEnumに変更し、型ヒントを強化
from enum import Enum

class CouponCode(str, Enum):
    SAVE10 = "SAVE10"
    HALF = "HALF"

DISCOUNT_RATES: dict[str, float] = {
    CouponCode.SAVE10: 0.10,
    CouponCode.HALF: 0.50,
}

def calculate_discount(price: float, coupon_code: str) -> float:
    if price < 0:
        raise ValueError(f"Price must be non-negative, got {price}")
    discount_rate = DISCOUNT_RATES.get(coupon_code, 0.0)
    return round(price * (1 - discount_rate), 2)
```

テストは変わらず全PASSED。

## Jest（JavaScript/TypeScript）での実例

```typescript
// userService.test.ts
import { UserService } from './userService';
import { mockDb } from './mocks/db';

describe('UserService', () => {
  let service: UserService;

  beforeEach(() => {
    service = new UserService(mockDb);
  });

  it('should create a user with hashed password', async () => {
    const user = await service.createUser('test@example.com', 'password123');
    expect(user.email).toBe('test@example.com');
    expect(user.passwordHash).not.toBe('password123');
    expect(user.passwordHash).toMatch(/^\$2[aby]\$/);  // bcrypt hash
  });

  it('should throw on duplicate email', async () => {
    await service.createUser('dup@example.com', 'pass');
    await expect(
      service.createUser('dup@example.com', 'pass2')
    ).rejects.toThrow('Email already exists');
  });
});
```

## モック生成を自動化する

Claude Codeに依存関係のモックも生成させます。

```bash
claude "以下のクラスのモックをJestで生成してください。
対象: src/repositories/UserRepository.ts
モック対象メソッド: findById, findByEmail, save, delete"
```

生成されるモック例：

```typescript
// mocks/UserRepository.ts
export const mockUserRepository = {
  findById: jest.fn().mockResolvedValue({
    id: '1',
    email: 'test@example.com',
    passwordHash: '$2b$10$xxx',
    createdAt: new Date('2026-01-01')
  }),
  findByEmail: jest.fn().mockResolvedValue(null),
  save: jest.fn().mockImplementation((user) => Promise.resolve(user)),
  delete: jest.fn().mockResolvedValue(true)
};
```

## JUnit（Java）でのTDD

```java
// DiscountServiceTest.java
@ExtendWith(MockitoExtension.class)
class DiscountServiceTest {

    @Mock
    private CouponRepository couponRepository;

    @InjectMocks
    private DiscountService discountService;

    @Test
    void testValidCouponApplied() {
        when(couponRepository.findByCode("SAVE10"))
            .thenReturn(Optional.of(new Coupon("SAVE10", 0.10)));

        double result = discountService.apply(1000.0, "SAVE10");

        assertEquals(900.0, result, 0.001);
    }

    @Test
    void testInvalidCouponNoDiscount() {
        when(couponRepository.findByCode("INVALID"))
            .thenReturn(Optional.empty());

        double result = discountService.apply(1000.0, "INVALID");

        assertEquals(1000.0, result, 0.001);
    }
}
```

## /test-genスキルの設定例

`.claude/commands/test-gen.md`:

```markdown
対象ファイルのテストコードを生成してください。

## ルール
1. テストファイルは `test_` プレフィックスまたは `.test.` を含む名前にする
2. 正常系・異常系・境界値を必ずカバーする
3. 外部依存（DB、API、ファイルI/O）はモックを使う
4. 各テストは独立していること（他のテストに依存しない）

## フォーマット
- Python: pytest + 型ヒント
- JavaScript/TypeScript: Jest + TypeScript
- Java: JUnit 5 + Mockito

## 出力
テストコードのみを出力する（説明不要）
```

## TDDの効果

Claude Codeでテスト生成を自動化したチームの実績：

| 指標 | 前 | 後 |
|------|----|----|
| テストカバレッジ | 23% | 78% |
| テスト作成時間 | 30分/関数 | 3分/関数 |
| バグ発見タイミング | リリース後 | 開発中 |
| リファクタ恐怖感 | 高 | 低 |

テストがあればリファクタが怖くない。リファクタが怖くなければコードが綺麗になる。コードが綺麗なら次のみょうが（AI）が読みやすくなる。正のサイクルです。

## まとめ

1. `/test-gen` スキルで関数からテストを自動生成
2. Red（失敗） → Green（通す） → Refactor（整える）のサイクルを守る
3. モック生成もClaudeに任せて外部依存を切り離す
4. Pytest・Jest・JUnit全てに対応

TDDの心理的ハードル「テストを書くのが面倒」はAIで消えます。

---

> **Code Review Pack（¥980）** — `/code-review`・`/test-gen`・`/refactor` の3スキルセット。TDD加速に必要なツールが全部入り。
>
> 👉 [note.com/myougatheaxo](https://note.com/myougatheaxo)
