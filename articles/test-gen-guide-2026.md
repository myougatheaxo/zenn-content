---
title: "Claude Codeで単体テストを自動生成する - /test-gen スキルの実践"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "テスト", "TDD", "Python", "Jest"]
published: true
---

## はじめに

「テストを書きたいけど時間がない」——開発現場でよく聞く言葉だ。

機能追加が優先され、テストは後回し。気づいたらカバレッジ30%台のまま本番に出ているコードが量産されている。テストを書く行為そのものが面倒というより、**何をテストすればいいか考える時間**が惜しいのが本音だろう。

Claude Codeの `/test-gen` スキルはそこを自動化する。ファイルを渡すだけで、Happy path・Edge case・Error case・Integration の4方向からテストを生成してくれる。

## /test-gen の仕組み — 4つの生成戦略

`/test-gen` は単純なコードの複製ではなく、以下の4つの観点からテストを生成する。

**1. Happy path（正常系）**
関数が期待通りの入力を受け取り、期待通りの出力を返すケース。最も基本的なテストで、実装の仕様確認にもなる。

**2. Edge cases（境界値）**
空文字・ゼロ・最大値・最小値・None など、通常の使い方の境界にあるケース。バグが潜みやすいのはここだ。

**3. Error cases（異常系）**
不正な型・範囲外の値・権限エラーなど、例外が発生することを期待するケース。`pytest.raises` や `expect(...).toThrow()` を使った検証が生成される。

**4. Integration（結合）**
外部依存（DB・API・ファイルシステム）をモックし、複数のコンポーネントの連携を検証するケース。単体テストでは見えない連携バグを拾う。

## Python/pytest での実際の出力例

例として、ユーザーサービスのメソッドにテストを生成してみよう。

**対象コード（`src/services/user.py`）:**

```python
class UserService:
    def __init__(self, db):
        self.db = db

    def create_user(self, name: str, email: str, age: int) -> dict:
        if not name or not email:
            raise ValueError("name and email are required")
        if age < 0 or age > 150:
            raise ValueError(f"invalid age: {age}")
        if "@" not in email:
            raise ValueError(f"invalid email: {email}")
        user = {"name": name, "email": email, "age": age}
        self.db.insert("users", user)
        return user
```

**実行コマンド:**

```bash
/test-gen src/services/user.py --coverage 80
```

**生成されるテスト（`tests/test_user_service.py`）:**

```python
import pytest
from unittest.mock import MagicMock
from src.services.user import UserService


@pytest.fixture
def mock_db():
    return MagicMock()


@pytest.fixture
def service(mock_db):
    return UserService(db=mock_db)


# --- Happy path ---
class TestCreateUserHappyPath:
    def test_valid_user_is_created(self, service, mock_db):
        result = service.create_user("Alice", "alice@example.com", 30)
        assert result == {"name": "Alice", "email": "alice@example.com", "age": 30}
        mock_db.insert.assert_called_once_with("users", result)

    def test_returns_dict_with_correct_keys(self, service):
        result = service.create_user("Bob", "bob@example.com", 25)
        assert set(result.keys()) == {"name", "email", "age"}


# --- Edge cases ---
class TestCreateUserEdgeCases:
    def test_age_zero_is_valid(self, service):
        result = service.create_user("Newborn", "baby@example.com", 0)
        assert result["age"] == 0

    def test_age_150_is_valid(self, service):
        result = service.create_user("Elder", "elder@example.com", 150)
        assert result["age"] == 150

    def test_single_char_name(self, service):
        result = service.create_user("A", "a@example.com", 20)
        assert result["name"] == "A"


# --- Error cases ---
class TestCreateUserErrors:
    def test_empty_name_raises_value_error(self, service):
        with pytest.raises(ValueError, match="required"):
            service.create_user("", "user@example.com", 25)

    def test_empty_email_raises_value_error(self, service):
        with pytest.raises(ValueError, match="required"):
            service.create_user("Alice", "", 25)

    def test_negative_age_raises_value_error(self, service):
        with pytest.raises(ValueError, match="invalid age"):
            service.create_user("Alice", "alice@example.com", -1)

    def test_age_over_150_raises_value_error(self, service):
        with pytest.raises(ValueError, match="invalid age"):
            service.create_user("Alice", "alice@example.com", 151)

    def test_email_without_at_raises_value_error(self, service):
        with pytest.raises(ValueError, match="invalid email"):
            service.create_user("Alice", "notanemail", 25)


# --- Integration ---
class TestCreateUserIntegration:
    def test_db_insert_called_with_correct_data(self, service, mock_db):
        service.create_user("Carol", "carol@example.com", 40)
        mock_db.insert.assert_called_once()
        args = mock_db.insert.call_args[0]
        assert args[0] == "users"
        assert args[1]["email"] == "carol@example.com"

    def test_db_error_propagates(self, service, mock_db):
        mock_db.insert.side_effect = RuntimeError("DB connection failed")
        with pytest.raises(RuntimeError, match="DB connection failed"):
            service.create_user("Dave", "dave@example.com", 35)
```

正常系・境界値・異常系・DB連携テストまで、1コマンドで生成された。

## JavaScript/Jest での出力例

フロントエンドでも同様に動作する。

**対象コード（`src/utils/price.js`）:**

```javascript
export function calculateDiscount(price, discountRate) {
  if (price < 0) throw new Error("price must be non-negative");
  if (discountRate < 0 || discountRate > 1) {
    throw new Error("discountRate must be between 0 and 1");
  }
  return Math.round(price * (1 - discountRate));
}
```

**実行コマンド:**

```bash
/test-gen src/utils/price.js
```

**生成されるテスト（`__tests__/price.test.js`）:**

```javascript
import { calculateDiscount } from '../src/utils/price';

describe('calculateDiscount', () => {
  // Happy path
  describe('正常系', () => {
    test('10%割引を正しく計算する', () => {
      expect(calculateDiscount(1000, 0.1)).toBe(900);
    });

    test('割引なし(0%)は元の価格を返す', () => {
      expect(calculateDiscount(500, 0)).toBe(500);
    });

    test('100%割引は0を返す', () => {
      expect(calculateDiscount(1000, 1)).toBe(0);
    });
  });

  // Edge cases
  describe('境界値', () => {
    test('price=0 は 0 を返す', () => {
      expect(calculateDiscount(0, 0.5)).toBe(0);
    });

    test('小数点以下は四捨五入される', () => {
      expect(calculateDiscount(100, 0.333)).toBe(67);
    });
  });

  // Error cases
  describe('異常系', () => {
    test('負の価格でエラーを投げる', () => {
      expect(() => calculateDiscount(-1, 0.1)).toThrow('price must be non-negative');
    });

    test('割引率が1より大きい場合にエラーを投げる', () => {
      expect(() => calculateDiscount(1000, 1.1)).toThrow('discountRate must be between 0 and 1');
    });

    test('割引率が負の場合にエラーを投げる', () => {
      expect(() => calculateDiscount(1000, -0.1)).toThrow('discountRate must be between 0 and 1');
    });
  });
});
```

## カバレッジ目標設定（80%+）

`--coverage` オプションでカバレッジ目標を指定すると、スキルがその水準を満たすよう追加ケースを生成する。

```bash
/test-gen src/services/user.py --coverage 80
```

80%を目標にしている理由は、100%を追いかけるコストに対してリターンが逓減するからだ。テストがなかった状態から80%に到達するだけで、バグ検出率は大幅に向上する。

| カバレッジ目標 | 生成されるケース数 | 用途 |
|------------|-------------|------|
| 60% | 最小限（Happy path + 主要エラー） | レガシー改修の足がかり |
| 80% | 標準（4戦略すべて） | 新規開発の基準値 |
| 95%+ | 網羅的（全分岐・全例外） | 金融・医療など高信頼性が必要な領域 |

## 実際の使い方

```bash
# 単一ファイル
/test-gen src/services/user.py

# カバレッジ目標付き
/test-gen src/services/user.py --coverage 80

# ディレクトリごと
/test-gen src/services/

# Jest向け（拡張子で自動判定）
/test-gen src/utils/price.js
```

生成されたテストファイルは即座に実行可能な状態で出力される。

```bash
# Python
pytest tests/test_user_service.py -v --cov=src

# JavaScript
npx jest __tests__/price.test.js --coverage
```

モックの設定・フィクスチャの定義まで含まれているため、そのまま実行してグリーンになることを確認するだけでいい。調整が必要な箇所は出力のコメントで明示される。

## まとめ

`/test-gen` は「テストを書く時間がない」を「テストを確認する時間に変換する」スキルだ。

- **4戦略**（Happy / Edge / Error / Integration）で網羅的なケースを自動生成
- Python/pytest・JavaScript/Jest どちらにも対応
- `--coverage` オプションで目標カバレッジに合わせて生成量を調整
- 生成されたテストはそのまま実行可能な形で出力される

テストを後回しにしてきたコードベースがあるなら、まず `/test-gen` を実行してみてほしい。

---

## Code Review Packについて

この記事で紹介した `/test-gen` は、**Code Review Pack**（¥980）に含まれています。

- `/code-review` — 5軸コードレビュー自動化
- `/refactor-suggest` — 技術的負債の自動検出
- `/test-gen` — 単体テスト自動生成

👉 [PromptWorksで購入する](https://prompt-works.jp)

*みょうが (@myougaTheAxo) — セキュリティ重視のClaudeエンジニア。*
