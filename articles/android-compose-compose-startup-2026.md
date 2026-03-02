---
title: "Compose Startup完全ガイド — App Startup/Initializer/遅延初期化/起動時間最適化"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "startup"]
published: true
---

## この記事で学べること

**Compose Startup**（App Startup、Initializer、遅延初期化、起動時間最適化）を解説します。

---

## App Startup

```groovy
dependencies {
    implementation("androidx.startup:startup-runtime:1.1.1")
}
```

```kotlin
// Timber初期化
class TimberInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// Coil初期化（Timber依存）
class CoilInitializer : Initializer<ImageLoader> {
    override fun create(context: Context): ImageLoader {
        Timber.d("Coil初期化")
        return ImageLoader.Builder(context)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .build()
            .also { Coil.setImageLoader(it) }
    }
    override fun dependencies(): List<Class<out Initializer<*>>> =
        listOf(TimberInitializer::class.java)
}
```

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
// 手動で遅延初期化
class AnalyticsInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        FirebaseAnalytics.getInstance(context)
    }
    override fun dependencies() = emptyList<Class<out Initializer<*>>>()
}

// 必要時に初期化
class AnalyticsHelper(private val context: Context) {
    fun initialize() {
        AppInitializer.getInstance(context)
            .initializeComponent(AnalyticsInitializer::class.java)
    }
}
```

```xml
<!-- 自動初期化を無効化 -->
<provider android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false" tools:node="merge">
    <meta-data android:name="com.example.app.AnalyticsInitializer"
        android:value="androidx.startup"
        tools:node="remove" />
</provider>
```

---

## 起動時間計測

```kotlin
class StartupMetrics {
    companion object {
        private var processStartTime = SystemClock.elapsedRealtime()

        fun measureFirstFrame(activity: Activity) {
            activity.window.decorView.viewTreeObserver.addOnPreDrawListener(
                object : ViewTreeObserver.OnPreDrawListener {
                    override fun onPreDraw(): Boolean {
                        activity.window.decorView.viewTreeObserver.removeOnPreDrawListener(this)
                        val elapsed = SystemClock.elapsedRealtime() - processStartTime
                        Timber.d("初回フレーム描画: ${elapsed}ms")
                        return true
                    }
                }
            )
        }
    }
}

@Composable
fun SplashScreen(onReady: () -> Unit) {
    LaunchedEffect(Unit) {
        // 最小限の初期化のみ
        delay(100)
        onReady()
    }

    Box(Modifier.fillMaxSize().background(MaterialTheme.colorScheme.primary),
        contentAlignment = Alignment.Center) {
        CircularProgressIndicator(color = Color.White)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Initializer` | 初期化タスク定義 |
| `dependencies` | 依存順序 |
| `AppInitializer` | 手動初期化 |
| `tools:node="remove"` | 自動初期化無効 |

- App Startupで初期化順序を宣言的に管理
- `dependencies()`で依存関係を定義
- `tools:node="remove"`で遅延初期化に切り替え
- ContentProvider統合で起動コスト削減

---

8種類のAndroidアプリテンプレート（最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose BaselineProfile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
- [Compose Benchmark](https://zenn.dev/myougatheaxo/articles/android-compose-compose-benchmark-2026)
- [Compose R8Optimization](https://zenn.dev/myougatheaxo/articles/android-compose-compose-r8-optimization-2026)
