---
title: "Kotlin委譲パターンガイド — by/lazy/observable/mapで実装効率化"
emoji: "🔄"
type: "tech"
topics: ["android", "kotlin", "delegation", "patterns"]
published: true
---

## この記事で学べること

Kotlinの**委譲（delegation）パターン**（by, lazy, observable, map）を解説します。

---

## クラス委譲 (by)

```kotlin
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) {
        println("[LOG] $message")
    }
}

// LoggerインターフェースをConsoleLoggerに委譲
class UserService(logger: Logger) : Logger by logger {
    fun createUser(name: String) {
        log("ユーザー作成: $name") // loggerに委譲
    }
}

// 使用例
val service = UserService(ConsoleLogger())
service.createUser("Alice") // [LOG] ユーザー作成: Alice
```

---

## プロパティ委譲: lazy

```kotlin
class ExpensiveRepository {
    // 初回アクセス時に1回だけ初期化
    val database by lazy {
        println("DB初期化中...")
        Database.connect()
    }

    // スレッドセーフでない場合（シングルスレッド限定）
    val cache by lazy(LazyThreadSafetyMode.NONE) {
        mutableMapOf<String, Any>()
    }
}
```

---

## プロパティ委譲: observable

```kotlin
import kotlin.properties.Delegates

class UserSettings {
    // 変更を監視
    var theme by Delegates.observable("light") { _, old, new ->
        println("テーマ変更: $old → $new")
        applyTheme(new)
    }

    // 変更を拒否可能
    var fontSize by Delegates.vetoable(14) { _, _, new ->
        new in 8..32 // 8-32の範囲のみ許可
    }

    // 初期値なし（初回代入前にアクセスするとエラー）
    var userName by Delegates.notNull<String>()
}

val settings = UserSettings()
settings.theme = "dark" // テーマ変更: light → dark
settings.fontSize = 100 // 拒否される、14のまま
```

---

## プロパティ委譲: map

```kotlin
// Mapからプロパティを委譲
class Config(map: Map<String, Any?>) {
    val host: String by map
    val port: Int by map
    val debug: Boolean by map
}

val config = Config(
    mapOf(
        "host" to "localhost",
        "port" to 8080,
        "debug" to true
    )
)

println(config.host) // localhost
println(config.port) // 8080

// MutableMapなら書き込みも可能
class MutableConfig(map: MutableMap<String, Any?>) {
    var host: String by map
    var port: Int by map
}
```

---

## カスタムデリゲート

```kotlin
import kotlin.reflect.KProperty

// SharedPreferencesデリゲート
class PreferenceDelegate<T>(
    private val prefs: SharedPreferences,
    private val key: String,
    private val defaultValue: T
) {
    @Suppress("UNCHECKED_CAST")
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return when (defaultValue) {
            is String -> prefs.getString(key, defaultValue) as T
            is Int -> prefs.getInt(key, defaultValue) as T
            is Boolean -> prefs.getBoolean(key, defaultValue) as T
            is Long -> prefs.getLong(key, defaultValue) as T
            is Float -> prefs.getFloat(key, defaultValue) as T
            else -> throw IllegalArgumentException("未対応の型")
        }
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        prefs.edit().apply {
            when (value) {
                is String -> putString(key, value)
                is Int -> putInt(key, value)
                is Boolean -> putBoolean(key, value)
                is Long -> putLong(key, value)
                is Float -> putFloat(key, value)
            }
            apply()
        }
    }
}

// 使用例
class AppSettings(prefs: SharedPreferences) {
    var username by PreferenceDelegate(prefs, "username", "")
    var darkMode by PreferenceDelegate(prefs, "dark_mode", false)
    var fontSize by PreferenceDelegate(prefs, "font_size", 14)
}
```

---

## Android実践: ViewModel委譲

```kotlin
// Activity/Fragmentで
class MainActivity : ComponentActivity() {
    // viewModels()はbyで委譲
    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MainScreen(viewModel)
        }
    }
}

// Hilt使用時
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    private val viewModel: MainViewModel by viewModels()
}
```

---

## まとめ

- **クラス委譲**: `class A : Interface by impl` でメソッド転送
- **lazy**: 初回アクセス時に初期化、以降キャッシュ
- **observable**: 値変更時にコールバック
- **vetoable**: 条件付きで変更を拒否
- **map**: Mapのキーをプロパティにマッピング
- **カスタム**: `getValue`/`setValue`演算子で自作デリゲート
- `by viewModels()`もKotlinの委譲パターン

---

8種類のAndroidアプリテンプレート（Kotlin最新パターン使用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [data classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-2026)
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
