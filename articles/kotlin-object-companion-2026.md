---
title: "Kotlin object/companion object完全ガイド — シングルトン/ファクトリ/定数"
emoji: "🏗️"
type: "tech"
topics: ["android", "kotlin", "design-pattern", "oop"]
published: true
---

## この記事で学べること

Kotlinの**object宣言**（シングルトン、companion object、object式、ファクトリパターン）を解説します。

---

## object宣言（シングルトン）

```kotlin
// スレッドセーフなシングルトン
object AppConfig {
    var apiBaseUrl = "https://api.example.com"
    var isDebug = false
    var maxRetries = 3

    fun configure(baseUrl: String, debug: Boolean) {
        apiBaseUrl = baseUrl
        isDebug = debug
    }
}

// 使用
AppConfig.configure("https://staging.api.example.com", true)
println(AppConfig.apiBaseUrl)

// ログユーティリティ
object Logger {
    fun d(tag: String, message: String) {
        if (AppConfig.isDebug) {
            Log.d(tag, message)
        }
    }

    fun e(tag: String, message: String, throwable: Throwable? = null) {
        Log.e(tag, message, throwable)
    }
}
```

---

## companion object

```kotlin
class User private constructor(
    val id: String,
    val name: String,
    val email: String
) {
    companion object {
        // 定数
        const val MAX_NAME_LENGTH = 50
        const val TABLE_NAME = "users"

        // ファクトリメソッド
        fun create(name: String, email: String): User {
            require(name.length <= MAX_NAME_LENGTH) { "Name too long" }
            return User(
                id = UUID.randomUUID().toString(),
                name = name,
                email = email
            )
        }

        // JSONから生成
        fun fromJson(json: JsonObject): User {
            return User(
                id = json.getString("id"),
                name = json.getString("name"),
                email = json.getString("email")
            )
        }
    }
}

// 使用
val user = User.create("Alice", "alice@example.com")
println(User.MAX_NAME_LENGTH) // 50
```

---

## companion objectの名前付き

```kotlin
class ApiClient {
    companion object Factory {
        fun create(config: ApiConfig): ApiClient {
            return ApiClient().apply {
                // 設定適用
            }
        }
    }
}

// 使用
val client = ApiClient.Factory.create(config)
// または
val client2 = ApiClient.create(config)
```

---

## companion objectでインターフェース実装

```kotlin
interface JsonParser<T> {
    fun fromJson(json: String): T
}

data class Product(val id: String, val name: String, val price: Int) {
    companion object : JsonParser<Product> {
        override fun fromJson(json: String): Product {
            val obj = JSONObject(json)
            return Product(
                id = obj.getString("id"),
                name = obj.getString("name"),
                price = obj.getInt("price")
            )
        }
    }
}

// 使用
val product = Product.fromJson("""{"id":"1","name":"Item","price":100}""")

// ジェネリック関数で活用
fun <T> loadFromCache(parser: JsonParser<T>, key: String): T? {
    val json = cache.get(key) ?: return null
    return parser.fromJson(json)
}

val product = loadFromCache(Product, "product_1")
```

---

## object式（匿名オブジェクト）

```kotlin
// インターフェース実装（1回限り）
val comparator = object : Comparator<User> {
    override fun compare(a: User, b: User): Int {
        return a.name.compareTo(b.name)
    }
}

// 複数インターフェース実装
val handler = object : ClickListener, LongClickListener {
    override fun onClick(view: View) { /* ... */ }
    override fun onLongClick(view: View): Boolean { /* ... */ return true }
}

// SAM変換（Javaインターフェース）
// object式の代わりにlambdaが使える
val listener = View.OnClickListener { view ->
    // 処理
}
```

---

## 定数定義パターン

```kotlin
// ❌ companion objectのval（ランタイム定数）
class Bad {
    companion object {
        val API_URL = "https://api.example.com" // getterが生成される
    }
}

// ✅ constでコンパイル時定数
class Good {
    companion object {
        const val API_URL = "https://api.example.com" // インライン化
    }
}

// ✅ トップレベル定数（Kotlinらしい）
const val API_URL = "https://api.example.com"

// ✅ objectで関連定数をグループ化
object Routes {
    const val HOME = "home"
    const val DETAIL = "detail/{id}"
    const val SETTINGS = "settings"

    fun detail(id: String) = "detail/$id"
}

object PrefsKeys {
    val THEME = stringPreferencesKey("theme")
    val LANGUAGE = stringPreferencesKey("language")
    val NOTIFICATIONS = booleanPreferencesKey("notifications")
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `object` | シングルトン、ユーティリティ |
| `companion object` | ファクトリ、定数、静的メソッド |
| `object : Interface` | 匿名実装（1回限り） |
| `const val` | コンパイル時定数 |

- `object`はスレッドセーフなシングルトン
- `companion object`はJavaの`static`相当
- `const val`はプリミティブ/Stringのみ
- ファクトリパターンで`private constructor`+`companion object`
- companion objectはインターフェース実装可能

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed class](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [data classパターン](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-patterns-2026)
- [スコープ関数](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
