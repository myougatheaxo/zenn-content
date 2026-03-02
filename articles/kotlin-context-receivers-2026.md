---
title: "Kotlin context parameters完全ガイド — 暗黙的コンテキスト/DSL設計"
emoji: "🔑"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "programming"]
published: true
---

## この記事で学べること

**context parameters**（暗黙的コンテキスト渡し、DSL設計、ログ/トランザクション/認証コンテキスト）を解説します。

---

## context parametersとは

Kotlin 2.1で実験的に導入。関数にコンテキストを暗黙的に渡せます。

```kotlin
// 有効化 (build.gradle.kts)
kotlin {
    compilerOptions {
        freeCompilerArgs.add("-Xcontext-parameters")
    }
}
```

---

## 基本

```kotlin
interface Logger {
    fun log(message: String)
}

// context(Logger) = Loggerスコープ内でのみ呼び出し可能
context(logger: Logger)
fun processData(data: String) {
    logger.log("Processing: $data")
    // 処理ロジック
    logger.log("Done: $data")
}

// 呼び出し側
class ConsoleLogger : Logger {
    override fun log(message: String) = println("[LOG] $message")
}

fun main() {
    with(ConsoleLogger()) {
        processData("sample")  // Loggerコンテキストが暗黙的に渡される
    }
}
```

---

## 複数コンテキスト

```kotlin
interface TransactionScope {
    suspend fun <T> execute(query: String): T
}

interface AuthContext {
    val currentUserId: String
    val isAdmin: Boolean
}

context(auth: AuthContext, tx: TransactionScope)
suspend fun deleteUser(userId: String) {
    require(auth.isAdmin) { "Admin required" }
    tx.execute<Unit>("DELETE FROM users WHERE id = '$userId'")
}
```

---

## Compose DSLとの組み合わせ

```kotlin
interface AnalyticsScope {
    fun trackEvent(name: String, params: Map<String, String> = emptyMap())
}

context(analytics: AnalyticsScope)
@Composable
fun TrackedButton(
    text: String,
    eventName: String,
    onClick: () -> Unit
) {
    Button(onClick = {
        analytics.trackEvent(eventName, mapOf("button" to text))
        onClick()
    }) {
        Text(text)
    }
}
```

---

## with/runによるコンテキスト提供

```kotlin
class AppAnalytics : AnalyticsScope {
    override fun trackEvent(name: String, params: Map<String, String>) {
        Firebase.analytics.logEvent(name) {
            params.forEach { (k, v) -> param(k, v) }
        }
    }
}

@Composable
fun MyScreen() {
    val analytics = remember { AppAnalytics() }
    with(analytics) {
        TrackedButton(
            text = "購入",
            eventName = "purchase_click",
            onClick = { /* 購入処理 */ }
        )
    }
}
```

---

## まとめ

| 概念 | 説明 |
|------|------|
| `context(T)` | 暗黙的コンテキスト宣言 |
| `with(instance)` | コンテキスト提供 |
| 複数コンテキスト | `context(A, B)` |
| DSL設計 | スコープ付き関数 |

- context parametersでボイラープレート削減
- Logger/Auth/Transactionなど横断的関心事に最適
- `with`/`run`でコンテキスト提供
- Kotlin 2.1以降で実験的機能

---

8種類のAndroidアプリテンプレート（最新Kotlin活用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スコープ関数](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [DSLビルダー](https://zenn.dev/myougatheaxo/articles/kotlin-dsl-builder-2026)
- [高階関数](https://zenn.dev/myougatheaxo/articles/kotlin-higher-order-functions-2026)
