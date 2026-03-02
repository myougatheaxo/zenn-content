---
title: "Hilt Multibinding完全ガイド — @IntoSet/@IntoMap/プラグインパターン"
emoji: "🔌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt Multibinding**（@IntoSet、@IntoMap、マルチバインディング、プラグインパターン）を解説します。

---

## @IntoSet

```kotlin
interface Initializer {
    fun initialize()
}

class LoggingInitializer @Inject constructor() : Initializer {
    override fun initialize() { Timber.plant(Timber.DebugTree()) }
}

class AnalyticsInitializer @Inject constructor() : Initializer {
    override fun initialize() { /* Analytics初期化 */ }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class InitializerModule {
    @Binds @IntoSet
    abstract fun bindLogging(impl: LoggingInitializer): Initializer

    @Binds @IntoSet
    abstract fun bindAnalytics(impl: AnalyticsInitializer): Initializer
}

// 使用: 全Initializerを一括実行
class AppBootstrap @Inject constructor(
    private val initializers: Set<@JvmSuppressWildcards Initializer>
) {
    fun run() { initializers.forEach { it.initialize() } }
}
```

---

## @IntoMap

```kotlin
// ViewModelキー
@MapKey
@Retention(AnnotationRetention.RUNTIME)
annotation class FeatureKey(val value: String)

interface Feature {
    fun getName(): String
    fun isEnabled(): Boolean
}

@Module
@InstallIn(SingletonComponent::class)
abstract class FeatureModule {
    @Binds @IntoMap @FeatureKey("dark_mode")
    abstract fun bindDarkMode(impl: DarkModeFeature): Feature

    @Binds @IntoMap @FeatureKey("push_notification")
    abstract fun bindPush(impl: PushFeature): Feature
}

// 使用
class FeatureManager @Inject constructor(
    private val features: Map<String, @JvmSuppressWildcards Feature>
) {
    fun isEnabled(key: String): Boolean = features[key]?.isEnabled() ?: false
    fun getAllFeatures(): Map<String, Feature> = features
}
```

---

## Compose表示

```kotlin
@Composable
fun FeatureToggleScreen(featureManager: FeatureManager) {
    val features = featureManager.getAllFeatures()

    LazyColumn(Modifier.padding(16.dp)) {
        items(features.entries.toList()) { (key, feature) ->
            ListItem(
                headlineContent = { Text(feature.getName()) },
                trailingContent = {
                    Switch(checked = feature.isEnabled(), onCheckedChange = { /* toggle */ })
                }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@IntoSet` | Set<T>に追加 |
| `@IntoMap` | Map<K,V>に追加 |
| `@MapKey` | カスタムマップキー |
| `@JvmSuppressWildcards` | ワイルドカード問題回避 |

- `@IntoSet`で複数実装をSet<T>に集約
- `@IntoMap`でキー付きで複数実装を管理
- プラグインパターンで拡張可能な設計
- `@JvmSuppressWildcards`はKotlinジェネリクスとJavaの互換性のため必要

---

8種類のAndroidアプリテンプレート（Hilt DI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Hilt Module](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-module-2026)
- [Hilt ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-viewmodel-2026)
- [Hilt Testing](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
