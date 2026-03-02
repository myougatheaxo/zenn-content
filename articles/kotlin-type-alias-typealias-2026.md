---
title: "Kotlin typealias/value class活用ガイド — 型安全の強化"
emoji: "🏷️"
type: "tech"
topics: ["android", "kotlin", "typesafety", "tips"]
published: true
---

## この記事で学べること

Kotlinの**typealias**と**value class**（型エイリアス、インライン型、型安全強化）を解説します。

---

## typealias

```kotlin
// 長い型名の省略
typealias UserMap = Map<String, List<User>>
typealias ClickHandler = (Int) -> Unit
typealias Predicate<T> = (T) -> Boolean

// Compose向け
typealias ComposableContent = @Composable () -> Unit
typealias OnItemClick<T> = (T) -> Unit

// 使用
fun processUsers(users: UserMap) { /* ... */ }

@Composable
fun ItemList(onItemClick: OnItemClick<Item>) {
    // ...
}
```

---

## value class（JVM inline class）

```kotlin
// ❌ 型安全でない: 文字列を取り違える
fun createUser(name: String, email: String, id: String) { }
// createUser("user@test.com", "Alice", "123") // コンパイル通る！

// ✅ value classで型安全に
@JvmInline
value class UserId(val value: String)

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email" }
    }
}

@JvmInline
value class UserName(val value: String) {
    init {
        require(value.isNotBlank()) { "Name cannot be blank" }
    }
}

fun createUser(name: UserName, email: Email, id: UserId) { }
// createUser(Email("test"), UserName("Alice"), UserId("1")) // コンパイルエラー！
// createUser(UserName("Alice"), Email("alice@test.com"), UserId("1")) // ✅ OK
```

---

## value class の実用例

```kotlin
// 金額（通貨を型で区別）
@JvmInline
value class Yen(val amount: Int) {
    operator fun plus(other: Yen) = Yen(amount + other.amount)
    operator fun times(multiplier: Int) = Yen(amount * multiplier)
}

@JvmInline
value class Dollar(val amount: Int)

fun charge(price: Yen) { /* ... */ }
// charge(Dollar(100)) // コンパイルエラー！型が違う

// パスワード（ログ出力防止）
@JvmInline
value class Password(val value: String) {
    override fun toString(): String = "****" // ログに出力されない
}

// ミリ秒（単位を型で表現）
@JvmInline
value class Milliseconds(val value: Long)

@JvmInline
value class Seconds(val value: Long) {
    fun toMilliseconds() = Milliseconds(value * 1000)
}
```

---

## typealiasとvalue classの使い分け

```kotlin
// typealias: 型の別名（コンパイル時に消える）
typealias UserId = String
// ↑ StringとUserIdは完全に同じ型。取り違え可能

// value class: 新しい型（コンパイル時チェックあり）
@JvmInline
value class UserId(val value: String)
// ↑ StringとUserIdは別の型。取り違え不可

// 判断基準:
// - 可読性のためだけ → typealias
// - 型安全が必要 → value class
// - ランタイムコスト → どちらもゼロ（inlineされる）
```

---

## まとめ

| 機能 | 目的 | ランタイムコスト |
|------|------|----------------|
| `typealias` | 型名の省略 | なし |
| `value class` | 新しい型の作成 | なし（inline） |

- `typealias`は複雑な型の可読性向上
- `value class`は型安全の強化（引数の取り違え防止）
- `value class`にはバリデーション（`init`ブロック）追加可能
- `toString()`オーバーライドでログ出力制御
- operator funで演算子オーバーロード対応

---

8種類のAndroidアプリテンプレート（型安全設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [ジェネリクスガイド](https://zenn.dev/myougatheaxo/articles/kotlin-generics-reified-2026)
- [data classパターン](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-patterns-2026)
