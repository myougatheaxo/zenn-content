---
title: "App Startupライブラリ + 起動最適化ガイド"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**App Startup**ライブラリとアプリ起動時間の最適化テクニックを解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.startup:startup-runtime:1.2.0")
}
```

---

## Initializer定義

```kotlin
class TimberInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

class CoilInitializer : Initializer<ImageLoader> {
    override fun create(context: Context): ImageLoader {
        return ImageLoader.Builder(context)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .build()
            .also { Coil.setImageLoader(it) }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

class AnalyticsInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        FirebaseAnalytics.getInstance(context)
    }

    // Timber初期化後に実行
    override fun dependencies() = listOf(TimberInitializer::class.java)
}
```

---

## AndroidManifest設定

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">

    <meta-data
        android:name="com.example.app.TimberInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="com.example.app.CoilInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="com.example.app.AnalyticsInitializer"
        android:value="androidx.startup" />
</provider>
```

---

## 起動時間最適化

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // Splash画面の制御
        val splashScreen = installSplashScreen()

        // 初期データ読み込み完了まで表示
        var isReady = false
        splashScreen.setKeepOnScreenCondition { !isReady }

        super.onCreate(savedInstanceState)
        enableEdgeToEdge()

        setContent {
            val viewModel: MainViewModel = hiltViewModel()
            val isLoaded by viewModel.isLoaded.collectAsStateWithLifecycle()

            LaunchedEffect(isLoaded) {
                if (isLoaded) isReady = true
            }

            MyAppTheme {
                if (isLoaded) {
                    AppNavigation()
                }
            }
        }
    }
}
```

---

## Baselineプロファイル

```kotlin
// benchmark/src/main/java/BaselineProfileGenerator.kt
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() {
        rule.collect(packageName = "com.example.app") {
            // コールドスタート
            pressHome()
            startActivityAndWait()

            // 主要画面へのナビゲーション
            device.findObject(By.text("Search")).click()
            device.waitForIdle()
        }
    }
}
```

---

## まとめ

- `App Startup`でContentProvider不要の初期化管理
- `dependencies()`で初期化順序を制御
- `SplashScreen`+`setKeepOnScreenCondition`でデータ待ち
- Baselineプロファイルで初回起動を高速化
- 重い処理は`lazy`や`viewModelScope`で遅延実行
- `enableEdgeToEdge()`を`super.onCreate()`の前に

---

8種類のAndroidアプリテンプレート（起動最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スプラッシュアニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-splash-animation-2026)
- [リコンポジション最適化](https://zenn.dev/myougatheaxo/articles/android-compose-recomposition-debug-2026)
- [Crashlytics/Analytics](https://zenn.dev/myougatheaxo/articles/android-compose-crash-analytics-2026)
