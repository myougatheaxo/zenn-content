---
title: "Kotlinプロパティ委譲ガイド — by lazy/observable/map"
emoji: "🎭"
type: "tech"
topics: ["android", "kotlin", "delegate", "property"]
published: true
---

## この記事で学べること

Kotlinの**プロパティ委譲**（by lazy、Delegates.observable、カスタムDelegate、Compose連携）を解説します。

---

## by lazy（遅延初期化）

```kotlin
// 初回アクセス時に1回だけ初期化
val heavyObject: ExpensiveObject by lazy {
    ExpensiveObject.create() // 初回のみ実行
}

// スレッドセーフモード
val config: Config by lazy(LazyThreadSafetyMode.SYNCHRONIZED) {
    loadConfig() // マルチスレッドでも安全
}

// シングルスレッド保証がある場合（高速）
val adapter: Adapter by lazy(LazyThreadSafetyMode.NONE) {
    Adapter(items) // UIスレッドでのみ使う場合
}

// Activityでの活用
class MainActivity : ComponentActivity() {
    private val viewModel: MainViewModel by viewModels()

    private val binding by lazy {
        ActivityMainBinding.inflate(layoutInflater)
    }
}
```

---

## Delegates.observable（変更監視）

```kotlin
import kotlin.properties.Delegates

class UserSettings {
    // 値が変更されるたびにコールバック
    var theme: String by Delegates.observable("light") { _, old, new ->
        println("Theme changed: $old → $new")
        applyTheme(new)
    }

    // 変更を拒否できる
    var fontSize: Int by Delegates.vetoable(16) { _, _, new ->
        new in 12..32 // 範囲外の値は拒否
    }

    // 初期化前のアクセスでエラー
    var apiKey: String by Delegates.notNull()
}
```

---

## Mapデリゲート

```kotlin
// Mapのキーをプロパティとしてアクセス
class Config(map: Map<String, Any?>) {
    val host: String by map
    val port: Int by map
    val debug: Boolean by map
}

val config = Config(mapOf(
    "host" to "localhost",
    "port" to 8080,
    "debug" to true
))

println(config.host) // "localhost"
println(config.port) // 8080

// MutableMapで書き込み可能
class MutableConfig(map: MutableMap<String, Any?>) {
    var host: String by map
    var port: Int by map
}
```

---

## カスタムDelegate

```kotlin
import kotlin.properties.ReadWriteProperty
import kotlin.reflect.KProperty

// SharedPreferences用Delegate
class PreferenceDelegate<T>(
    private val prefs: SharedPreferences,
    private val key: String,
    private val defaultValue: T
) : ReadWriteProperty<Any?, T> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return when (defaultValue) {
            is String -> prefs.getString(key, defaultValue as String) as T
            is Int -> prefs.getInt(key, defaultValue as Int) as T
            is Boolean -> prefs.getBoolean(key, defaultValue as Boolean) as T
            is Long -> prefs.getLong(key, defaultValue as Long) as T
            is Float -> prefs.getFloat(key, defaultValue as Float) as T
            else -> throw IllegalArgumentException("Unsupported type")
        }
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
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

// 使用
class AppPreferences(prefs: SharedPreferences) {
    var username by PreferenceDelegate(prefs, "username", "")
    var darkMode by PreferenceDelegate(prefs, "dark_mode", false)
    var fontSize by PreferenceDelegate(prefs, "font_size", 16)
}

val prefs = AppPreferences(sharedPrefs)
prefs.username = "Alice" // 自動保存
println(prefs.username) // 自動読み込み
```

---

## ログ付きDelegate

```kotlin
class LoggingDelegate<T>(private var value: T) : ReadWriteProperty<Any?, T> {
    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        Log.d("Delegate", "Get ${property.name} = $value")
        return value
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        Log.d("Delegate", "Set ${property.name}: ${this.value} → $value")
        this.value = value
    }
}

fun <T> logged(initialValue: T) = LoggingDelegate(initialValue)

// 使用
var counter by logged(0)
counter = 5  // Log: Set counter: 0 → 5
println(counter) // Log: Get counter = 5
```

---

## Composeでの委譲

```kotlin
// by remember { mutableStateOf() }
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    // count は Int として直接使える（delegateが getValue/setValue を実装）

    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// by collectAsStateWithLifecycle()
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    // users は List<User> として直接使える
}
```

---

## まとめ

| Delegate | 用途 |
|----------|------|
| `by lazy` | 遅延初期化（1回のみ） |
| `Delegates.observable` | 変更時コールバック |
| `Delegates.vetoable` | 変更の条件付き承認 |
| `Delegates.notNull` | 初期化前アクセス禁止 |
| `by map` | Mapキー→プロパティ |
| `ReadWriteProperty` | カスタム委譲 |

- `by`キーワードでgetter/setterを委譲
- `lazy`はスレッドセーフモード選択可能
- カスタムDelegateでSharedPreferences等をプロパティ化
- Composeの`by remember`も委譲パターン

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スコープ関数](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [拡張関数](https://zenn.dev/myougatheaxo/articles/kotlin-extension-functions-2026)
- [高階関数](https://zenn.dev/myougatheaxo/articles/kotlin-higher-order-functions-2026)
