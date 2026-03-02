---
title: "Firebase Crashlytics完全ガイド — クラッシュレポート/カスタムログ/非致命的エラー"
emoji: "🔥"
type: "tech"
topics: ["android", "firebase", "kotlin", "debugging"]
published: true
---

## この記事で学べること

**Firebase Crashlytics**（クラッシュレポート、カスタムキー、非致命的エラー、ユーザー識別、Compose統合）を解説します。

---

## 初期設定

```kotlin
// build.gradle.kts (project)
plugins {
    id("com.google.firebase.crashlytics") version "3.0.2" apply false
}

// build.gradle.kts (app)
plugins {
    id("com.google.firebase.crashlytics")
}

dependencies {
    implementation(platform(libs.firebase.bom))
    implementation("com.google.firebase:firebase-crashlytics-ktx")
}
```

---

## カスタムキー/ログ

```kotlin
class CrashlyticsHelper @Inject constructor() {
    private val crashlytics = Firebase.crashlytics

    fun setUser(userId: String) {
        crashlytics.setUserId(userId)
    }

    fun setCustomKeys(keys: Map<String, Any>) {
        crashlytics.setCustomKeys {
            keys.forEach { (key, value) ->
                when (value) {
                    is String -> key(key, value)
                    is Int -> key(key, value)
                    is Boolean -> key(key, value)
                    is Float -> key(key, value)
                }
            }
        }
    }

    fun log(message: String) {
        crashlytics.log(message)
    }

    fun recordNonFatal(exception: Exception) {
        crashlytics.recordException(exception)
    }
}
```

---

## 非致命的エラー記録

```kotlin
class SafeApiCall @Inject constructor(
    private val crashlytics: CrashlyticsHelper
) {
    suspend fun <T> execute(
        tag: String,
        block: suspend () -> T
    ): Result<T> {
        return try {
            crashlytics.log("API call: $tag")
            Result.success(block())
        } catch (e: IOException) {
            crashlytics.setCustomKeys(mapOf("api_tag" to tag, "error_type" to "network"))
            crashlytics.recordNonFatal(e)
            Result.failure(e)
        } catch (e: HttpException) {
            crashlytics.setCustomKeys(mapOf("api_tag" to tag, "http_code" to e.code()))
            crashlytics.recordNonFatal(e)
            Result.failure(e)
        }
    }
}
```

---

## Compose ErrorBoundary連携

```kotlin
@Composable
fun CrashReportingContent(content: @Composable () -> Unit) {
    val crashlytics = Firebase.crashlytics

    CompositionLocalProvider(
        LocalCrashlyticsLogger provides crashlytics
    ) {
        // Compose内のエラーをキャッチ
        ErrorBoundary(
            onError = { error ->
                crashlytics.log("Compose error in ${error.component}")
                crashlytics.recordException(error.exception)
            }
        ) {
            content()
        }
    }
}
```

---

## デバッグ設定

```kotlin
// Application
@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // デバッグビルドではCrashlyticsを無効化
        Firebase.crashlytics.setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)
    }
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| カスタムキー | `setCustomKeys` |
| ログ | `log()` |
| 非致命的エラー | `recordException()` |
| ユーザー識別 | `setUserId()` |

- 自動でクラッシュスタックトレースを収集
- `setCustomKeys`でコンテキスト情報を付与
- `recordException`で非致命的エラーも記録
- デバッグビルドでは`setCrashlyticsCollectionEnabled(false)`

---

8種類のAndroidアプリテンプレート（Crashlytics設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Analytics](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-analytics-2026)
- [Firebase Performance](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-performance-2026)
- [エラーUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
