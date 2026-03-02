---
title: "Kotlin DSL/ビルダーパターンガイド — apply/buildList/カスタムDSL"
emoji: "🔧"
type: "tech"
topics: ["android", "kotlin", "dsl", "patterns"]
published: true
---

## この記事で学べること

Kotlinの**DSLとビルダーパターン**（apply、buildList、カスタムDSL設計）を解説します。

---

## applyパターン

```kotlin
// ビルダーとしてのapply
val notification = NotificationCompat.Builder(context, channelId).apply {
    setContentTitle("タイトル")
    setContentText("メッセージ")
    setSmallIcon(R.drawable.ic_notification)
    setPriority(NotificationCompat.PRIORITY_DEFAULT)
    setAutoCancel(true)
}.build()

// オブジェクト初期化
data class User(
    var name: String = "",
    var email: String = "",
    var age: Int = 0
)

val user = User().apply {
    name = "Alice"
    email = "alice@example.com"
    age = 28
}
```

---

## buildListパターン

```kotlin
// 条件付きリスト構築
fun getMenuItems(isAdmin: Boolean, isLoggedIn: Boolean): List<MenuItem> = buildList {
    add(MenuItem("ホーム", Icons.Default.Home))
    add(MenuItem("検索", Icons.Default.Search))

    if (isLoggedIn) {
        add(MenuItem("プロフィール", Icons.Default.Person))
        add(MenuItem("設定", Icons.Default.Settings))
    }

    if (isAdmin) {
        add(MenuItem("管理画面", Icons.Default.AdminPanelSettings))
    }

    add(MenuItem("ヘルプ", Icons.Default.Help))
}

// buildMap
val config = buildMap {
    put("theme", "dark")
    put("language", "ja")
    put("fontSize", "16")
    if (BuildConfig.DEBUG) {
        put("debug", "true")
    }
}
```

---

## カスタムDSL: HTML風

```kotlin
class HtmlBuilder {
    private val elements = mutableListOf<String>()

    fun h1(text: String) { elements.add("<h1>$text</h1>") }
    fun p(text: String) { elements.add("<p>$text</p>") }
    fun ul(block: UlBuilder.() -> Unit) {
        val builder = UlBuilder()
        builder.block()
        elements.add(builder.build())
    }

    fun build(): String = elements.joinToString("\n")
}

class UlBuilder {
    private val items = mutableListOf<String>()
    fun li(text: String) { items.add("  <li>$text</li>") }
    fun build(): String = "<ul>\n${items.joinToString("\n")}\n</ul>"
}

fun html(block: HtmlBuilder.() -> Unit): String {
    return HtmlBuilder().apply(block).build()
}

// 使用
val page = html {
    h1("Kotlinガイド")
    p("DSLで構造化データを記述できます")
    ul {
        li("簡潔")
        li("型安全")
        li("IDEサポート")
    }
}
```

---

## カスタムDSL: UI設定

```kotlin
class FormDsl {
    val fields = mutableListOf<FieldConfig>()

    fun text(name: String, block: FieldConfig.() -> Unit = {}) {
        fields.add(FieldConfig(name, FieldType.TEXT).apply(block))
    }

    fun email(name: String, block: FieldConfig.() -> Unit = {}) {
        fields.add(FieldConfig(name, FieldType.EMAIL).apply(block))
    }

    fun password(name: String, block: FieldConfig.() -> Unit = {}) {
        fields.add(FieldConfig(name, FieldType.PASSWORD).apply(block))
    }

    fun number(name: String, block: FieldConfig.() -> Unit = {}) {
        fields.add(FieldConfig(name, FieldType.NUMBER).apply(block))
    }
}

data class FieldConfig(
    val name: String,
    val type: FieldType,
    var label: String = name,
    var required: Boolean = false,
    var maxLength: Int = Int.MAX_VALUE,
    var placeholder: String = ""
)

enum class FieldType { TEXT, EMAIL, PASSWORD, NUMBER }

fun form(block: FormDsl.() -> Unit): List<FieldConfig> {
    return FormDsl().apply(block).fields
}

// 使用
val loginForm = form {
    email("email") {
        label = "メールアドレス"
        required = true
        placeholder = "user@example.com"
    }
    password("password") {
        label = "パスワード"
        required = true
        maxLength = 100
    }
}
```

---

## @DslMarker

```kotlin
@DslMarker
annotation class ConfigDsl

@ConfigDsl
class AppConfig {
    var appName: String = ""
    private var _network: NetworkConfig? = null
    val network: NetworkConfig get() = _network!!

    fun network(block: NetworkConfig.() -> Unit) {
        _network = NetworkConfig().apply(block)
    }
}

@ConfigDsl
class NetworkConfig {
    var baseUrl: String = ""
    var timeout: Long = 30000
    var retryCount: Int = 3
}

fun appConfig(block: AppConfig.() -> Unit): AppConfig {
    return AppConfig().apply(block)
}

val config = appConfig {
    appName = "MyApp"
    network {
        baseUrl = "https://api.example.com"
        timeout = 15000
        retryCount = 5
    }
}
```

---

## まとめ

- `apply`でオブジェクト初期化パターン
- `buildList`/`buildMap`で条件付きコレクション構築
- レシーバー付きラムダ(`T.() -> Unit`)でDSL設計
- `@DslMarker`でスコープ制御
- ネストされたDSLで階層構造を表現
- 型安全でIDEの補完が効くAPI設計

---

8種類のAndroidアプリテンプレート（DSLパターン活用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [拡張関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-extension-functions-2026)
- [delegation/委譲ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-delegation-2026)
