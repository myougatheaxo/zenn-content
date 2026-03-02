---
title: "Kotlin Delegation完全ガイド — by lazy/by map/プロパティ委譲/クラス委譲"
emoji: "🤝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "delegation"]
published: true
---

## この記事で学べること

**Kotlin Delegation**（by lazy、プロパティ委譲、クラス委譲、カスタムDelegate）を解説します。

---

## プロパティ委譲

```kotlin
// by lazy: 初回アクセス時に初期化
class AppConfig {
    val database: AppDatabase by lazy {
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
    }

    val preferences: SharedPreferences by lazy {
        context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    }
}

// by map: Mapからプロパティを読み取り
class User(map: Map<String, Any?>) {
    val name: String by map
    val age: Int by map
    val email: String by map
}

val user = User(mapOf("name" to "太郎", "age" to 25, "email" to "taro@example.com"))
println(user.name) // 太郎
```

---

## カスタムDelegate

```kotlin
// SharedPreferencesのDelegate
class PreferenceDelegate<T>(
    private val prefs: SharedPreferences,
    private val key: String,
    private val defaultValue: T
) : ReadWriteProperty<Any?, T> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(thisRef: Any?, property: KProperty<*>): T = with(prefs) {
        when (defaultValue) {
            is String -> getString(key, defaultValue) as T
            is Int -> getInt(key, defaultValue) as T
            is Boolean -> getBoolean(key, defaultValue) as T
            is Long -> getLong(key, defaultValue) as T
            else -> throw IllegalArgumentException("Unsupported type")
        }
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) = with(prefs.edit()) {
        when (value) {
            is String -> putString(key, value)
            is Int -> putInt(key, value)
            is Boolean -> putBoolean(key, value)
            is Long -> putLong(key, value)
            else -> throw IllegalArgumentException("Unsupported type")
        }
        apply()
    }
}

// 使用
class Settings(prefs: SharedPreferences) {
    var darkMode: Boolean by PreferenceDelegate(prefs, "dark_mode", false)
    var userName: String by PreferenceDelegate(prefs, "user_name", "")
    var fontSize: Int by PreferenceDelegate(prefs, "font_size", 14)
}
```

---

## クラス委譲

```kotlin
// インターフェースの実装を別クラスに委譲
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println("[LOG] $message")
}

// byでLoggerの実装をConsoleLoggerに委譲
class Repository(logger: Logger) : Logger by logger {
    fun fetchData() {
        log("データ取得開始") // Logger.log()が呼ばれる
    }
}
```

---

## まとめ

| 委譲パターン | 用途 |
|-------------|------|
| `by lazy` | 遅延初期化 |
| `by map` | Mapからプロパティ |
| `by ReadWriteProperty` | カスタム読み書き |
| クラス委譲 | 実装の委譲 |

- `by lazy`はスレッドセーフな遅延初期化
- カスタムDelegateでSharedPreferences等を簡潔に
- クラス委譲でデコレーターパターンを簡潔に実現
- Composeの`by remember`/`by mutableStateOf`も委譲

---

8種類のAndroidアプリテンプレート（Kotlin対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin SealedClass](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-sealed-class-2026)
- [Kotlin ValueClass](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-value-class-2026)
- [Kotlin ExtensionFunction](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-extension-function-2026)
