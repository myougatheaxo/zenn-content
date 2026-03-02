---
title: "Kotlin高階関数ガイド — lambda/inline/crossinline/noinline"
emoji: "🔗"
type: "tech"
topics: ["android", "kotlin", "functional", "lambda"]
published: true
---

## この記事で学べること

Kotlinの**高階関数**（lambda式、inline関数、crossinline/noinline、型エイリアス）を解説します。

---

## 高階関数の基本

```kotlin
// 関数を引数に取る
fun <T> List<T>.customFilter(predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) result.add(item)
    }
    return result
}

// 使用
val adults = users.customFilter { it.age >= 18 }

// 関数を返す
fun multiplier(factor: Int): (Int) -> Int {
    return { number -> number * factor }
}

val double = multiplier(2)
println(double(5)) // 10
```

---

## lambda式パターン

```kotlin
// 最後の引数がlambdaなら括弧の外に出せる
list.filter { it > 0 }

// 引数が1つのlambdaは it で参照
list.map { it.toString() }

// 複数引数
map.forEach { (key, value) -> println("$key=$value") }

// 使わない引数は _
list.forEachIndexed { _, item -> println(item) }

// SAM変換（Javaインターフェース）
button.setOnClickListener { view ->
    // Kotlin lambda → View.OnClickListener自動変換
}
```

---

## inline関数

```kotlin
// inline: lambdaのオーバーヘッド除去（オブジェクト生成なし）
inline fun <T> measureTime(block: () -> T): Pair<T, Long> {
    val start = System.nanoTime()
    val result = block()
    val elapsed = System.nanoTime() - start
    return result to elapsed
}

// inlineならlambda内でreturnできる（non-local return）
inline fun <T> List<T>.findFirst(predicate: (T) -> Boolean): T? {
    for (item in this) {
        if (predicate(item)) return item // 呼び出し元のreturn
    }
    return null
}
```

---

## crossinline / noinline

```kotlin
// crossinline: non-local returnを禁止
inline fun runOnUiThread(crossinline block: () -> Unit) {
    Handler(Looper.getMainLooper()).post {
        block() // ここでreturnされると困るのでcrossinline
    }
}

// noinline: 特定のlambdaをインライン化しない
inline fun process(
    transform: (String) -> String,
    noinline callback: (String) -> Unit // 変数として保持する必要がある場合
) {
    val result = transform("input")
    someAsyncFunction(callback) // noinlineなら渡せる
}
```

---

## 型エイリアス

```kotlin
// 関数型に名前を付ける
typealias Predicate<T> = (T) -> Boolean
typealias Mapper<T, R> = (T) -> R
typealias EventHandler = (Event) -> Unit
typealias OnClick = () -> Unit

// 使用
fun <T> List<T>.filterBy(predicate: Predicate<T>): List<T> {
    return filter(predicate)
}

// Compose での活用
typealias ComposableContent = @Composable () -> Unit

@Composable
fun AppCard(
    title: ComposableContent,
    content: ComposableContent
) {
    Card {
        title()
        content()
    }
}
```

---

## 実践パターン

```kotlin
// ビルダーパターン
inline fun buildString(block: StringBuilder.() -> Unit): String {
    return StringBuilder().apply(block).toString()
}

val html = buildString {
    append("<html>")
    append("<body>Hello</body>")
    append("</html>")
}

// リソース管理（use パターン）
inline fun <T : Closeable, R> T.safeUse(block: (T) -> R): R? {
    return try {
        block(this)
    } catch (e: Exception) {
        null
    } finally {
        try { close() } catch (_: Exception) {}
    }
}

// 条件付き実行
inline fun <T> T.applyIf(condition: Boolean, block: T.() -> T): T {
    return if (condition) block() else this
}

// Modifier拡張
fun Modifier.conditionalPadding(addPadding: Boolean): Modifier {
    return applyIf(addPadding) { padding(16.dp) }
}
```

---

## まとめ

| キーワード | 用途 |
|-----------|------|
| `(T) -> R` | 関数型（引数T、戻り値R） |
| `inline` | lambdaのオーバーヘッド除去 |
| `crossinline` | non-local return禁止 |
| `noinline` | インライン化しない |
| `typealias` | 関数型に名前を付ける |

- `inline`で頻繁に呼ばれるユーティリティを最適化
- `crossinline`は別コルーチン/スレッドで実行するlambda用
- `typealias`で複雑な関数型を読みやすく
- レシーバ付きlambda（`T.() -> R`）でDSL構築

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スコープ関数](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [コレクション操作](https://zenn.dev/myougatheaxo/articles/kotlin-collection-operations-2026)
- [拡張関数](https://zenn.dev/myougatheaxo/articles/kotlin-extension-functions-2026)
