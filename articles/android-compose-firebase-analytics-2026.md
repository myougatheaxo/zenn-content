---
title: "Firebase Analytics完全ガイド — イベント計測/ユーザープロパティ/Compose連携"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Firebase Analytics**（カスタムイベント、ユーザープロパティ、画面トラッキング、Compose統合）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
    implementation("com.google.firebase:firebase-analytics-ktx")
}
```

---

## AnalyticsHelper

```kotlin
class AnalyticsHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val firebaseAnalytics = Firebase.analytics

    fun logEvent(name: String, params: Map<String, Any> = emptyMap()) {
        firebaseAnalytics.logEvent(name) {
            params.forEach { (key, value) ->
                when (value) {
                    is String -> param(key, value)
                    is Long -> param(key, value)
                    is Double -> param(key, value)
                    is Int -> param(key, value.toLong())
                }
            }
        }
    }

    fun logScreenView(screenName: String, screenClass: String) {
        firebaseAnalytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
            param(FirebaseAnalytics.Param.SCREEN_NAME, screenName)
            param(FirebaseAnalytics.Param.SCREEN_CLASS, screenClass)
        }
    }

    fun setUserProperty(name: String, value: String) {
        firebaseAnalytics.setUserProperty(name, value)
    }

    fun setUserId(userId: String?) {
        firebaseAnalytics.setUserId(userId)
    }
}
```

---

## 画面トラッキング

```kotlin
@Composable
fun TrackScreenView(screenName: String, analytics: AnalyticsHelper) {
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME) {
                analytics.logScreenView(screenName, screenName)
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}

@Composable
fun HomeScreen(analytics: AnalyticsHelper = hiltViewModel<MainViewModel>().analytics) {
    TrackScreenView("HomeScreen", analytics)

    Column(Modifier.padding(16.dp)) {
        Text("ホーム", style = MaterialTheme.typography.headlineMedium)
    }
}
```

---

## カスタムイベント

```kotlin
// ViewModel でイベント送信
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val analytics: AnalyticsHelper
) : ViewModel() {

    fun onProductClick(productId: String, productName: String) {
        analytics.logEvent("product_click", mapOf(
            "product_id" to productId,
            "product_name" to productName
        ))
    }

    fun onPurchaseComplete(amount: Int, currency: String) {
        analytics.logEvent(FirebaseAnalytics.Event.PURCHASE, mapOf(
            FirebaseAnalytics.Param.VALUE to amount.toLong(),
            FirebaseAnalytics.Param.CURRENCY to currency
        ))
    }

    fun onSearchPerformed(query: String, resultCount: Int) {
        analytics.logEvent(FirebaseAnalytics.Event.SEARCH, mapOf(
            FirebaseAnalytics.Param.SEARCH_TERM to query,
            "result_count" to resultCount.toLong()
        ))
    }
}
```

---

## ユーザープロパティ

```kotlin
fun onUserLogin(user: User) {
    analytics.setUserId(user.id)
    analytics.setUserProperty("account_type", user.accountType.name)
    analytics.setUserProperty("preferred_language", user.language)
}

fun onUserLogout() {
    analytics.setUserId(null)
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| イベント計測 | `logEvent()` |
| 画面追跡 | `SCREEN_VIEW` |
| ユーザー属性 | `setUserProperty()` |
| ユーザーID | `setUserId()` |

- `DisposableEffect`で画面表示を自動トラッキング
- カスタムイベントでユーザー行動を計測
- ユーザープロパティでセグメント分析
- Firebase Consoleでリアルタイム確認

---

8種類のAndroidアプリテンプレート（Analytics組み込み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Remote Config](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-remote-config-2026)
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [Crashlytics](https://zenn.dev/myougatheaxo/articles/android-compose-crash-analytics-2026)
