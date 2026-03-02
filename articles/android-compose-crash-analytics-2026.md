---
title: "Firebase Crashlytics + Analytics連携ガイド"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Firebase Crashlytics**と**Analytics**をComposeアプリに統合する方法を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.0.0"))
    implementation("com.google.firebase:firebase-crashlytics-ktx")
    implementation("com.google.firebase:firebase-analytics-ktx")
}
```

---

## Crashlytics設定

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        // デバッグビルドではCrashlyticsを無効化
        FirebaseCrashlytics.getInstance().apply {
            setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)
        }
    }
}
```

---

## カスタムログ/ユーザー情報

```kotlin
object CrashReporter {
    private val crashlytics = FirebaseCrashlytics.getInstance()

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
        crashlytics.recordException(e)
    }
}

// 使用
CrashReporter.setUserId("user_123")
CrashReporter.setCustomKey("screen", "HomeScreen")

try {
    riskyOperation()
} catch (e: Exception) {
    CrashReporter.log("Failed to perform risky operation")
    CrashReporter.recordException(e)
}
```

---

## Analytics イベント送信

```kotlin
object AnalyticsTracker {
    private val analytics = Firebase.analytics

    fun trackScreenView(screenName: String) {
        analytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
            param(FirebaseAnalytics.Param.SCREEN_NAME, screenName)
        }
    }

    fun trackButtonClick(buttonName: String, screen: String) {
        analytics.logEvent("button_click") {
            param("button_name", buttonName)
            param("screen", screen)
        }
    }

    fun trackPurchase(itemId: String, price: Double) {
        analytics.logEvent(FirebaseAnalytics.Event.PURCHASE) {
            param(FirebaseAnalytics.Param.ITEM_ID, itemId)
            param(FirebaseAnalytics.Param.VALUE, price)
            param(FirebaseAnalytics.Param.CURRENCY, "JPY")
        }
    }
}
```

---

## Composeでの画面トラッキング

```kotlin
@Composable
fun TrackScreen(screenName: String) {
    DisposableEffect(screenName) {
        AnalyticsTracker.trackScreenView(screenName)
        CrashReporter.setCustomKey("current_screen", screenName)
        onDispose {}
    }
}

// 各画面で使用
@Composable
fun HomeScreen() {
    TrackScreen("HomeScreen")

    Column {
        Button(onClick = {
            AnalyticsTracker.trackButtonClick("create_task", "HomeScreen")
        }) {
            Text("タスク作成")
        }
    }
}
```

---

## まとめ

- `FirebaseCrashlytics`でクラッシュレポート自動送信
- `recordException`で非致命的エラーも記録
- `setUserId`/`setCustomKey`でコンテキスト情報追加
- `Firebase.analytics.logEvent`でカスタムイベント送信
- `DisposableEffect`で画面表示トラッキング
- デバッグビルドではCrashlytics無効化推奨

---

8種類のAndroidアプリテンプレート（分析/監視対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [Cloud Firestore](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-firestore-2026)
- [FCMプッシュ通知](https://zenn.dev/myougatheaxo/articles/android-compose-fcm-push-2026)
