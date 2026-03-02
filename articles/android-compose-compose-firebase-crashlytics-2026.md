---
title: "Compose Firebase Crashlytics完全ガイド — クラッシュレポート/カスタムログ/非致命エラー"
emoji: "🔥"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose Firebase Crashlytics**（クラッシュレポート、カスタムログ、非致命エラー、ユーザー識別）を解説します。

---

## セットアップ

```groovy
// build.gradle (project)
plugins {
    id("com.google.firebase.crashlytics") version "3.0.2" apply false
}

// build.gradle (app)
plugins {
    id("com.google.firebase.crashlytics")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.1.0"))
    implementation("com.google.firebase:firebase-crashlytics-ktx")
}
```

---

## Crashlytics統合

```kotlin
class CrashReporter @Inject constructor() {
    private val crashlytics = Firebase.crashlytics

    fun setUserId(userId: String) {
        crashlytics.setUserId(userId)
    }

    fun log(message: String) {
        crashlytics.log(message)
    }

    fun setCustomKey(key: String, value: String) {
        crashlytics.setCustomKey(key, value)
    }

    fun recordException(e: Throwable) {
        crashlytics.recordException(e) // 非致命エラー
    }
}

// ViewModel内で使用
@HiltViewModel
class DataViewModel @Inject constructor(
    private val repository: DataRepository,
    private val crashReporter: CrashReporter
) : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            try {
                crashReporter.log("Data loading started")
                val data = repository.getData()
                crashReporter.log("Data loaded: ${data.size} items")
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                crashReporter.recordException(e) // 非致命エラーとして記録
                _uiState.value = UiState.Error(e.message ?: "不明なエラー")
            }
        }
    }
}
```

---

## Compose連携

```kotlin
// グローバル例外ハンドラー
class MyApp : Application() {
    @Inject lateinit var crashReporter: CrashReporter

    override fun onCreate() {
        super.onCreate()

        // Coroutine未処理例外
        val defaultHandler = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
            Firebase.crashlytics.log("Uncaught exception in ${thread.name}")
            defaultHandler?.uncaughtException(thread, throwable)
        }
    }
}

// 画面遷移ログ
@Composable
fun CrashlyticsNavHost(crashReporter: CrashReporter) {
    val navController = rememberNavController()

    LaunchedEffect(navController) {
        navController.currentBackStackEntryFlow.collect { entry ->
            val route = entry.destination.route ?: return@collect
            crashReporter.log("Screen: $route")
            crashReporter.setCustomKey("current_screen", route)
        }
    }

    NavHost(navController, startDestination = "home") {
        composable("home") { HomeScreen() }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `recordException` | 非致命エラー記録 |
| `log` | カスタムログ |
| `setUserId` | ユーザー識別 |
| `setCustomKey` | カスタムキー |

- `recordException()`で非致命エラーをFirebaseに送信
- `log()`で操作履歴を記録（クラッシュ時に表示）
- `setUserId()`でユーザー別にクラッシュを追跡
- ProGuardの`mapping.txt`をアップロードして難読化解除

---

8種類のAndroidアプリテンプレート（Crashlytics対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FirebaseAnalytics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-analytics-2026)
- [Compose ProGuard](https://zenn.dev/myougatheaxo/articles/android-compose-compose-proguard-2026)
- [Coroutine ExceptionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-handler-2026)
