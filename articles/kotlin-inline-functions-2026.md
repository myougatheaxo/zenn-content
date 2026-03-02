---
title: "Kotlin inline関数完全ガイド — crossinline/noinline/reified/契約"
emoji: "🔧"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "programming"]
published: true
---

## この記事で学べること

**inline関数**（crossinline、noinline、reified型パラメータ、コントラクト、パフォーマンス最適化）を解説します。

---

## inline関数とは

```kotlin
// inline = 呼び出し箇所にコードが展開される（関数オブジェクト生成なし）
inline fun <T> measureTime(block: () -> T): Pair<T, Long> {
    val start = System.nanoTime()
    val result = block()
    val elapsed = System.nanoTime() - start
    return result to elapsed
}

// 使用（lambdaオブジェクトが生成されない）
val (result, time) = measureTime {
    expensiveOperation()
}
```

---

## noinline

```kotlin
// noinline = このlambdaだけインライン化しない（変数に保存したい場合）
inline fun execute(
    action: () -> Unit,
    noinline onComplete: () -> Unit  // 変数に保存可能
) {
    action()
    val callback = onComplete  // ✅ noinlineなので代入可能
    callback()
}
```

---

## crossinline

```kotlin
// crossinline = non-local returnを禁止（別の実行コンテキストで呼ばれるlambda）
inline fun runOnUiThread(crossinline action: () -> Unit) {
    Handler(Looper.getMainLooper()).post {
        action()  // postのlambda内で呼ばれるためcrossinline必須
    }
}

// crossinlineなしだと↓のreturnがrunOnUiThread呼び出し元を抜けてしまう
runOnUiThread {
    // return  // ❌ コンパイルエラー（crossinlineのため）
    return@runOnUiThread  // ✅ ラベル付きreturnはOK
}
```

---

## reified型パラメータ

```kotlin
// reified = 実行時に型情報にアクセス可能（inlineでのみ使用可）
inline fun <reified T> Intent.getParcelableExtraCompat(key: String): T? {
    return if (Build.VERSION.SDK_INT >= 33) {
        getParcelableExtra(key, T::class.java)
    } else {
        @Suppress("DEPRECATION")
        getParcelableExtra(key) as? T
    }
}

// 使用
val user = intent.getParcelableExtraCompat<User>("user")

// JSONパース
inline fun <reified T> String.fromJson(): T {
    return Json.decodeFromString<T>(this)
}

val user = jsonString.fromJson<User>()
```

---

## 実践パターン

```kotlin
// ログ付き実行
inline fun <T> loggedExecution(tag: String, block: () -> T): T {
    Log.d(tag, "Start")
    return try {
        block().also { Log.d(tag, "Success") }
    } catch (e: Exception) {
        Log.e(tag, "Error", e)
        throw e
    }
}

// リソース安全解放
inline fun <T : Closeable, R> T.useWith(block: (T) -> R): R {
    return try {
        block(this)
    } finally {
        close()
    }
}

// 条件付き実行
inline fun <T> T.applyIf(condition: Boolean, block: T.() -> Unit): T {
    if (condition) block()
    return this
}

// Modifier拡張
fun Modifier.applyIf(condition: Boolean, modifier: Modifier.() -> Modifier): Modifier {
    return if (condition) modifier() else this
}
```

---

## まとめ

| 修飾子 | 用途 |
|--------|------|
| `inline` | lambda生成を避ける |
| `noinline` | lambdaを変数に保存 |
| `crossinline` | non-local return禁止 |
| `reified` | 実行時型情報アクセス |

- `inline`でlambdaのオブジェクト生成を回避
- `reified`でジェネリクスの型情報を実行時に利用
- `crossinline`で別コンテキストのlambdaを安全に
- 高頻度呼び出しの小さな関数に効果的

---

8種類のAndroidアプリテンプレート（Kotlin best practices適用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [value class](https://zenn.dev/myougatheaxo/articles/kotlin-value-class-inline-2026)
- [スコープ関数](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [高階関数](https://zenn.dev/myougatheaxo/articles/kotlin-higher-order-functions-2026)
