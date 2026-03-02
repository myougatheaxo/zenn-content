---
title: "Compose AppStartup完全ガイド — 起動最適化/Initializer/遅延初期化"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Compose AppStartup**（App Startup Library、Initializer、遅延初期化、起動時間最適化）を解説します。

---

## 基本Initializer

```kotlin
// build.gradle
// implementation "androidx.startup:startup-runtime:1.1.1"

class TimberInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
        Timber.d("Timber initialized")
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

class CoilInitializer : Initializer<ImageLoader> {
    override fun create(context: Context): ImageLoader {
        return ImageLoader.Builder(context)
            .crossfade(true)
            .memoryCache { MemoryCache.Builder(context).maxSizePercent(0.25).build() }
            .build()
            .also { Coil.setImageLoader(it) }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = listOf(TimberInitializer::class.java)
}
```

---

## AndroidManifest設定

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.app.CoilInitializer"
        android:value="androidx.startup" />
</provider>
```

---

## 遅延初期化

```kotlin
class AnalyticsInitializer : Initializer<Analytics> {
    override fun create(context: Context): Analytics {
        return Analytics.Builder(context)
            .setEnabled(!BuildConfig.DEBUG)
            .build()
    }

    override fun dependencies() = emptyList<Class<out Initializer<*>>>()
}

// 手動で遅延初期化
class MyApplication : Application() {
    val analytics: Analytics by lazy {
        AppInitializer.getInstance(this)
            .initializeComponent(AnalyticsInitializer::class.java)
    }
}

// 自動初期化を無効にして手動制御
// <meta-data
//     android:name="com.example.app.AnalyticsInitializer"
//     tools:node="remove" />
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Initializer<T>` | 初期化コンポーネント |
| `dependencies()` | 依存関係定義 |
| `AppInitializer` | 手動初期化 |
| `tools:node="remove"` | 自動初期化無効 |

- App Startupで`ContentProvider`乱立を防止
- `dependencies()`で初期化順序を制御
- 不要な初期化は遅延化で起動時間短縮
- `tools:node="remove"`で自動初期化を無効化

---

8種類のAndroidアプリテンプレート（起動最適化対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose BaselineProfile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
- [Compose Benchmark](https://zenn.dev/myougatheaxo/articles/android-compose-compose-benchmark-2026)
- [Compose SplashScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-splash-screen-2026)
