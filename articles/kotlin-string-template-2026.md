---
title: "Kotlin文字列操作ガイド — テンプレート/正規表現/マルチライン"
emoji: "📝"
type: "tech"
topics: ["android", "kotlin", "string", "basics"]
published: true
---

## この記事で学べること

Kotlinの**文字列操作**（テンプレート、正規表現、マルチライン、便利な拡張関数）を解説します。

---

## 文字列テンプレート

```kotlin
val name = "Alice"
val age = 28

// 基本
println("名前: $name, 年齢: $age")

// 式
println("来年は${age + 1}歳")

// プロパティアクセス
val list = listOf(1, 2, 3)
println("要素数: ${list.size}")

// 条件式
println("${if (age >= 20) "成人" else "未成年"}")
```

---

## マルチライン（raw string）

```kotlin
val json = """
    {
        "name": "$name",
        "age": $age
    }
""".trimIndent()

val sql = """
    SELECT * FROM users
    WHERE age >= 20
    AND active = 1
    ORDER BY name ASC
""".trimIndent()

// marginの削除
val text = """
    |1. 最初の項目
    |2. 次の項目
    |3. 最後の項目
""".trimMargin()
```

---

## 便利な拡張関数

```kotlin
// null/空チェック
val input: String? = null
input.isNullOrEmpty()    // true
input.isNullOrBlank()    // true
"  ".isBlank()           // true
"  ".isEmpty()           // false

// 変換
"hello".uppercase()      // HELLO
"HELLO".lowercase()      // hello
"hello world".replaceFirstChar { it.uppercase() } // Hello world

// 分割・結合
"a,b,c".split(",")      // [a, b, c]
listOf("a", "b", "c").joinToString(", ") // a, b, c

// トリム
"  hello  ".trim()       // hello
"  hello  ".trimStart()  // "hello  "
"  hello  ".trimEnd()    // "  hello"

// 部分文字列
"Hello World".substringBefore(" ")  // Hello
"Hello World".substringAfter(" ")   // World
"Hello World".take(5)               // Hello
"Hello World".takeLast(5)           // World

// パディング
"42".padStart(5, '0')    // 00042
"hi".padEnd(10, '.')     // hi........
```

---

## 正規表現

```kotlin
// マッチ確認
val emailRegex = Regex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}")
emailRegex.matches("user@example.com")    // true
emailRegex.containsMatchIn("email: user@example.com") // true

// 値の抽出
val dateRegex = Regex("(\\d{4})-(\\d{2})-(\\d{2})")
val match = dateRegex.find("2026-03-02")
match?.groupValues    // [2026-03-02, 2026, 03, 02]
val (year, month, day) = match!!.destructured

// 置換
val masked = "電話: 090-1234-5678".replace(Regex("\\d{4}-\\d{4}$"), "****-****")
// 電話: 090-****-****

// 全一致を取得
val numbers = Regex("\\d+").findAll("a1b2c3").map { it.value }.toList()
// [1, 2, 3]
```

---

## フォーマット

```kotlin
// 数値フォーマット
val price = 1234567
"¥%,d".format(price)               // ¥1,234,567
"%.2f".format(3.14159)              // 3.14
"%05d".format(42)                   // 00042

// 日時フォーマット
val now = LocalDateTime.now()
val formatter = DateTimeFormatter.ofPattern("yyyy年MM月dd日 HH:mm")
now.format(formatter)               // 2026年03月02日 15:30

// BuildString
val html = buildString {
    appendLine("<html>")
    appendLine("  <body>")
    appendLine("    <h1>$name</h1>")
    appendLine("  </body>")
    appendLine("</html>")
}
```

---

## Android実践: 文字列リソース

```kotlin
// strings.xml
// <string name="greeting">こんにちは、%1$s さん！</string>
// <plurals name="item_count">
//     <item quantity="one">%d item</item>
//     <item quantity="other">%d items</item>
// </plurals>

@Composable
fun FormattedText() {
    val name = "Alice"
    Text(stringResource(R.string.greeting, name))
    // こんにちは、Alice さん！

    val count = 5
    Text(pluralStringResource(R.plurals.item_count, count, count))
    // 5 items
}
```

---

## まとめ

- `"$変数"` / `"${式}"`で文字列テンプレート
- `""" """.trimIndent()`でマルチライン
- `isNullOrBlank()`でnull+空白チェック
- `Regex`でパターンマッチ・抽出・置換
- `"%,d".format()`で数値フォーマット
- `buildString`で効率的な文字列構築
- `stringResource`でローカライズ対応

---

8種類のAndroidアプリテンプレート（多言語対応設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキスト装飾ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-text-styling-2026)
- [多言語対応ガイド](https://zenn.dev/myougatheaxo/articles/android-localization-i18n-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
