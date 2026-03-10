---
title: "Accelerate TDD with Claude Code: Auto-Generate Tests Using /test-gen Skill"
emoji: "🧪"
type: "tech"
topics: ["ClaudeCode", "TDD", "Python", "JavaScript", "Testing"]
published: false
---

## What is TDD and Why Does It Pair Well with AI?

TDD (Test-Driven Development) is a development approach that follows the Red -> Green -> Refactor cycle:

1. **Red**: Write a failing test first
2. **Green**: Write the minimum code to make it pass
3. **Refactor**: Clean up the code

The biggest barrier to TDD is the psychological cost of "writing tests is tedious." That's exactly where Claude Code helps.

## Basic Usage of the /test-gen Skill

After writing a function, immediately pass it to `/test-gen`:

```bash
claude /test-gen src/utils/calculator.py
```

Or, generate tests before writing the implementation (pure TDD):

```bash
claude "Write tests for the following function first.
Function: calculate_discount(price: float, coupon_code: str) -> float
Spec: 'SAVE10' gives 10% off, 'HALF' gives 50% off, invalid code gives 0% off"
```

## Real Example with Pytest (Python)

### Step 1: Red (Write the Failing Test)

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

Since `calculator.py` doesn't exist yet, all tests fail (Red).

### Step 2: Green (Write the Implementation)

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
# All tests PASSED (Green)
```

### Step 3: Refactor

```python
# After refactor: use Enum for constants, stronger type hints
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

All tests still PASSED.

## Real Example with Jest (JavaScript/TypeScript)

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

## Auto-Generate Mocks

Let Claude Code generate dependency mocks too:

```bash
claude "Generate Jest mocks for the following class.
Target: src/repositories/UserRepository.ts
Methods to mock: findById, findByEmail, save, delete"
```

Generated mock example:

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

## TDD with JUnit (Java)

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

## /test-gen Skill Configuration

`.claude/commands/test-gen.md`:

```markdown
Generate test code for the target file.

## Rules
1. Test file names use `test_` prefix or include `.test.`
2. Always cover happy path, error cases, and edge cases
3. Use mocks for external dependencies (DB, API, file I/O)
4. Each test must be independent

## Format
- Python: pytest with type hints
- JavaScript/TypeScript: Jest with TypeScript
- Java: JUnit 5 with Mockito

## Output
Output test code only (no explanation needed)
```

## TDD Impact

Results from a team that automated test generation with Claude Code:

| Metric | Before | After |
|--------|--------|-------|
| Test coverage | 23% | 78% |
| Time to write tests | 30 min/function | 3 min/function |
| Bug discovery timing | Post-release | During development |
| Fear of refactoring | High | Low |

Tests eliminate fear of refactoring. No fear of refactoring means cleaner code. Cleaner code means the next developer (or AI) can read it faster. A positive cycle.

## Summary

1. Use `/test-gen` skill to auto-generate tests from functions
2. Follow the Red (fail) -> Green (pass) -> Refactor (clean) cycle
3. Let Claude generate mocks too, to isolate external dependencies
4. Works with Pytest, Jest, and JUnit

The psychological barrier to TDD — "writing tests is tedious" — disappears with AI.

---

> **Code Review Pack** — A 3-skill set: `/code-review`, `/test-gen`, `/refactor`. Everything you need to accelerate TDD.
>
> 👉 [note.com/myougatheaxo](https://note.com/myougatheaxo)
