---
title: "Kotlin文字列処理ガイド — テンプレート/正規表現/変換"
emoji: "🔤"
type: "tech"
topics: ["android", "kotlin", "string", "tips"]
published: true
---

## この記事で学べること

Kotlinの**文字列処理**（テンプレート、正規表現、変換、マルチライン）を解説します。

---

## 文字列テンプレート

```kotlin
val name = "Kotlin"
val version = 2.0

// 基本
val greeting = "Hello, $name!"
val info = "Version: ${version + 0.1}"

// 条件式
val status = "User is ${if (isActive) "active" else "inactive"}"

// 関数呼び出し
val upper = "Name: ${name.uppercase()}"
```

---

## Raw String（複数行）

```kotlin
val json = """
    {
        "name": "Alice",
        "age": 25,
        "email": "alice@example.com"
    }
""".trimIndent()

val sql = """
    SELECT *
    FROM users
    WHERE active = 1
    ORDER BY name ASC
""".trimIndent()

// マージン除去
val text = """
    |First line
    |Second line
    |Third line
""".trimMargin()
```

---

## 正規表現

```kotlin
// 基本マッチ
val emailRegex = Regex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}")
val isValid = emailRegex.matches("user@example.com") // true

// 検索
val text = "Price: $100, Discount: $20"
val prices = Regex("\\$(\\d+)").findAll(text)
    .map { it.groupValues[1].toInt() }
    .toList() // [100, 20]

// 置換
val cleaned = "Hello  World   Kotlin".replace(Regex("\\s+"), " ")
// "Hello World Kotlin"

// 分解
val (year, month, day) = Regex("(\\d{4})-(\\d{2})-(\\d{2})")
    .matchEntire("2026-03-02")!!
    .destructured
```

---

## 変換/フォーマット

```kotlin
// 大文字/小文字
"hello".uppercase()          // "HELLO"
"HELLO".lowercase()          // "hello"
"hello world".replaceFirstChar { it.uppercase() } // "Hello world"

// パディング
"42".padStart(5, '0')        // "00042"
"Hi".padEnd(10, '.')         // "Hi........"

// 数値フォーマット
val price = 1234567
val formatted = "%,d".format(price)  // "1,234,567"
val decimal = "%.2f".format(3.14159) // "3.14"

// URLエンコード
val encoded = URLEncoder.encode("日本語", "UTF-8")
```

---

## 便利な拡張関数

```kotlin
// null/空チェック
val name: String? = null
name.isNullOrEmpty()   // true
name.isNullOrBlank()   // true
name.orEmpty()         // ""

// 分割
"a,b,c".split(",")           // ["a", "b", "c"]
"one.two.three".substringBefore(".")  // "one"
"one.two.three".substringAfter(".")   // "two.three"
"one.two.three".substringAfterLast(".") // "three"

// 変換
"123".toIntOrNull()           // 123
"abc".toIntOrNull()           // null
"true".toBooleanStrictOrNull() // true

// 繰り返し
"Ha".repeat(3) // "HaHaHa"
```

---

## まとめ

- `$variable`/`${expression}`で文字列テンプレート
- `""" """.trimIndent()`でマルチライン文字列
- `Regex`でパターンマッチング/置換/分解
- `padStart`/`padEnd`でパディング
- `toIntOrNull()`で安全な型変換
- `isNullOrBlank()`でnull安全な空チェック

---

8種類のAndroidアプリテンプレート（Kotlin最適設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [コレクション操作ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-collection-operations-2026)
- [スコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [data classパターン](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-patterns-2026)
