---
title: "Kotlin DSL完全ガイド — ビルダーパターン/型安全/レシーバー付きラムダ/Gradle DSL"
emoji: "🏗️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "dsl"]
published: true
---

## この記事で学べること

**Kotlin DSL**（ビルダーパターン、型安全DSL、レシーバー付きラムダ、Gradle Kotlin DSL）を解説します。

---

## レシーバー付きラムダ

```kotlin
// DSLの基本: レシーバー付きラムダ
class HtmlBuilder {
    private val elements = mutableListOf<String>()

    fun h1(text: String) { elements.add("<h1>$text</h1>") }
    fun p(text: String) { elements.add("<p>$text</p>") }
    fun ul(block: UlBuilder.() -> Unit) {
        val builder = UlBuilder().apply(block)
        elements.add(builder.build())
    }

    fun build(): String = elements.joinToString("\n")
}

class UlBuilder {
    private val items = mutableListOf<String>()
    fun li(text: String) { items.add("<li>$text</li>") }
    fun build(): String = "<ul>\n${items.joinToString("\n")}\n</ul>"
}

fun html(block: HtmlBuilder.() -> Unit): String =
    HtmlBuilder().apply(block).build()

// 使用
val page = html {
    h1("Kotlin DSL")
    p("型安全なDSLを構築できます")
    ul {
        li("簡潔")
        li("型安全")
        li("IDE補完対応")
    }
}
```

---

## 型安全ビルダー

```kotlin
@DslMarker
annotation class NetworkDsl

@NetworkDsl
class ApiClientBuilder {
    var baseUrl: String = ""
    var timeout: Long = 30_000
    private var headers = mutableMapOf<String, String>()
    private var interceptors = mutableListOf<(String) -> String>()

    fun headers(block: HeaderBuilder.() -> Unit) {
        HeaderBuilder(headers).apply(block)
    }

    fun interceptor(block: (String) -> String) {
        interceptors.add(block)
    }

    fun build(): ApiClient = ApiClient(baseUrl, timeout, headers, interceptors)
}

@NetworkDsl
class HeaderBuilder(private val headers: MutableMap<String, String>) {
    infix fun String.to(value: String) { headers[this] = value }
}

fun apiClient(block: ApiClientBuilder.() -> Unit): ApiClient =
    ApiClientBuilder().apply(block).build()

// 使用
val client = apiClient {
    baseUrl = "https://api.example.com"
    timeout = 10_000
    headers {
        "Authorization" to "Bearer token"
        "Accept" to "application/json"
    }
    interceptor { request -> request.also { println("Request: $it") } }
}
```

---

## Compose風DSL

```kotlin
@DslMarker
annotation class FormDsl

@FormDsl
class FormBuilder {
    private val fields = mutableListOf<FormField>()

    fun text(name: String, block: TextFieldBuilder.() -> Unit = {}) {
        fields.add(TextFieldBuilder(name).apply(block).build())
    }

    fun number(name: String, block: NumberFieldBuilder.() -> Unit = {}) {
        fields.add(NumberFieldBuilder(name).apply(block).build())
    }

    fun build(): List<FormField> = fields.toList()
}

fun form(block: FormBuilder.() -> Unit): List<FormField> =
    FormBuilder().apply(block).build()

// 使用
val loginForm = form {
    text("email") {
        label = "メールアドレス"
        required = true
        validator = { it.contains("@") }
    }
    text("password") {
        label = "パスワード"
        required = true
        minLength = 8
    }
}
```

---

## まとめ

| 機能 | 用途 |
|------|------|
| レシーバー付きラムダ | DSLの基本構造 |
| `@DslMarker` | スコープ制御 |
| `apply` | ビルダーパターン |
| `infix` | 自然な記法 |

- レシーバー付きラムダでDSLの構文を定義
- `@DslMarker`で外側スコープの関数を誤って呼ぶのを防止
- `apply`でビルダーにブロックを適用
- Compose自体がKotlin DSLの最良の実例

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin ScopeFunction](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-scope-function-2026)
- [Kotlin ExtensionFunction](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-extension-function-2026)
- [Kotlin Delegation](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-delegation-2026)
