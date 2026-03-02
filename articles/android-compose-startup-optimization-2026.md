---
title: "起動高速化完全ガイド — App Startup/遅延初期化/トレース分析"
emoji: "⚡"
type: "tech"
topics: ["android", "kotlin", "performance", "startup"]
published: true
---

## この記事で学べること

**起動高速化**（App Startup、遅延初期化、Baseline Profile、Macroトレース分析）を解説します。

---

## App Startup

```kotlin
// Initializer定義
class WorkManagerInitializer : Initializer<WorkManager> {
    override fun create(context: Context): WorkManager {
        val config = Configuration.Builder().build()
        WorkManager.initialize(context, config)
        return WorkManager.getInstance(context)
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

class AnalyticsInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        FirebaseApp.initializeApp(context)
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
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
        android:name="com.example.WorkManagerInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="com.example.AnalyticsInitializer"
        android:value="androidx.startup" />
</provider>
```

---

## 遅延初期化

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    // ❌ 起動時に全て初期化
    // val heavyLib = HeavyLibrary.init(this)

    // ✅ 遅延初期化
    val heavyLib by lazy { HeavyLibrary.init(this) }

    override fun onCreate() {
        super.onCreate()

        // ❌ UIスレッドで重い処理
        // database.migrate()

        // ✅ バックグラウンドで実行
        CoroutineScope(Dispatchers.IO).launch {
            database.migrate()
        }
    }
}
```

---

## スプラッシュ中のプリロード

```kotlin
@HiltViewModel
class StartupViewModel @Inject constructor(
    private val userRepo: UserRepository,
    private val configRepo: ConfigRepository
) : ViewModel() {
    val isReady = MutableStateFlow(false)

    init {
        viewModelScope.launch {
            // 並行で初期化
            supervisorScope {
                val user = async { userRepo.preload() }
                val config = async { configRepo.preload() }

                try {
                    user.await()
                    config.await()
                } catch (e: Exception) {
                    // エラーでも起動は続行
                }
            }
            isReady.value = true
        }
    }
}
```

---

## 起動時間計測

```kotlin
// Macrobenchmark
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun startupCold() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }

    @Test
    fun startupWarm() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.WARM
    ) {
        pressHome()
        startActivityAndWait()
    }
}
```

---

## チェックリスト

```kotlin
// 起動高速化チェックリスト:
// ✅ Application.onCreate を軽量化
// ✅ 重い初期化は遅延（lazy）or バックグラウンド
// ✅ App Startup で初期化順序を制御
// ✅ Baseline Profile で AOT コンパイル
// ✅ SplashScreen でプリロード
// ✅ ContentProvider の自動初期化を無効化
// ❌ UIスレッドでDB/ネットワーク操作
// ❌ 起動時に全ライブラリを初期化
```

---

## まとめ

| 手法 | 効果 |
|------|------|
| App Startup | 初期化順序の最適化 |
| 遅延初期化 | 起動時の処理削減 |
| Baseline Profile | AOTコンパイルで高速化 |
| 並行プリロード | 待機時間の最小化 |

- `Initializer`で初期化の順序と依存関係を管理
- `lazy`と`Dispatchers.IO`で重い処理を後回し
- Baseline Profileでコールドスタート30%高速化
- Macrobenchmarkで起動時間を定量的に計測

---

8種類のAndroidアプリテンプレート（起動最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Macrobenchmark](https://zenn.dev/myougatheaxo/articles/android-compose-benchmark-macro-2026)
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-baseline-profile-2026)
- [パフォーマンス](https://zenn.dev/myougatheaxo/articles/android-compose-stability-performance-2026)
