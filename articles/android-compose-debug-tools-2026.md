---
title: "Android開発デバッグツール完全ガイド — Layout Inspector/Profiler/LeakCanary"
emoji: "🐛"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "debugging"]
published: true
---

## この記事で学べること

**デバッグツール**（Layout Inspector、Compose Profiler、LeakCanary、StrictMode、OkHttp Logging）を解説します。

---

## Layout Inspector

Android StudioのLayout InspectorはCompose UIの階層構造をリアルタイムで表示します。

```kotlin
// デバッグビルドでInspector情報を有効化
android {
    buildTypes {
        debug {
            // Layout Inspectorが自動的にCompose情報を表示
        }
    }
}
```

---

## LeakCanary

```kotlin
// build.gradle.kts
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
}

// 設定不要！debugビルドで自動的にメモリリーク検出
// リーク検出時に通知が表示される
```

---

## StrictMode

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .penaltyLog()
                    .build()
            )
            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedClosableObjects()
                    .detectActivityLeaks()
                    .penaltyLog()
                    .build()
            )
        }
    }
}
```

---

## OkHttp Logging

```kotlin
// build.gradle.kts
dependencies {
    debugImplementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().apply {
            if (BuildConfig.DEBUG) {
                addInterceptor(HttpLoggingInterceptor().apply {
                    level = HttpLoggingInterceptor.Level.BODY
                })
            }
        }.build()
    }
}
```

---

## Compose再コンポジション可視化

```kotlin
// デバッグ用Modifier
fun Modifier.recompositionCounter(): Modifier = composed {
    val count = remember { mutableIntStateOf(0) }
    count.intValue++

    if (BuildConfig.DEBUG) {
        drawWithContent {
            drawContent()
            drawRect(
                color = Color.Red.copy(alpha = 0.1f * minOf(count.intValue, 10)),
                size = size
            )
        }
    } else this
}

// 使用
Text("Hello", Modifier.recompositionCounter())
```

---

## Timber（構造化ログ）

```kotlin
dependencies {
    implementation("com.jakewharton.timber:timber:5.0.1")
}

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        } else {
            Timber.plant(CrashlyticsTree())
        }
    }
}

class CrashlyticsTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority >= Log.ERROR) {
            Firebase.crashlytics.log(message)
            t?.let { Firebase.crashlytics.recordException(it) }
        }
    }
}
```

---

## まとめ

| ツール | 用途 |
|--------|------|
| Layout Inspector | UIレイアウト確認 |
| LeakCanary | メモリリーク検出 |
| StrictMode | パフォーマンス違反検出 |
| HTTP Logging | API通信デバッグ |
| Timber | 構造化ログ |

- `debugImplementation`でリリースに含めない
- LeakCanaryは導入するだけで自動検出
- StrictModeでメインスレッドの違反を検出
- Timberでrelease/debugのログ出力を制御

---

8種類のAndroidアプリテンプレート（デバッグ設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [再コンポジションデバッグ](https://zenn.dev/myougatheaxo/articles/android-compose-recomposition-debug-2026)
- [パフォーマンス](https://zenn.dev/myougatheaxo/articles/android-compose-performance-profiling-2026)
- [Crashlytics](https://zenn.dev/myougatheaxo/articles/android-compose-crash-analytics-2026)
