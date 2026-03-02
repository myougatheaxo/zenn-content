---
title: "Compose Firebase Analytics完全ガイド — イベント計測/画面追跡/ユーザープロパティ"
emoji: "📈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose Firebase Analytics**（イベントログ、画面追跡、ユーザープロパティ、Composeとの統合）を解説します。

---

## セットアップ

```groovy
// build.gradle
plugins {
    id("com.google.gms.google-services")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.1.0"))
    implementation("com.google.firebase:firebase-analytics-ktx")
}
```

---

## イベント計測

```kotlin
class AnalyticsHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val analytics = Firebase.analytics

    fun logEvent(name: String, params: Map<String, Any> = emptyMap()) {
        analytics.logEvent(name) {
            params.forEach { (key, value) ->
                when (value) {
                    is String -> param(key, value)
                    is Long -> param(key, value)
                    is Double -> param(key, value)
                }
            }
        }
    }

    fun logScreenView(screenName: String) {
        analytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
            param(FirebaseAnalytics.Param.SCREEN_NAME, screenName)
        }
    }

    fun logPurchase(itemName: String, price: Double) {
        analytics.logEvent(FirebaseAnalytics.Event.PURCHASE) {
            param(FirebaseAnalytics.Param.ITEM_NAME, itemName)
            param(FirebaseAnalytics.Param.VALUE, price)
            param(FirebaseAnalytics.Param.CURRENCY, "JPY")
        }
    }

    fun setUserProperty(key: String, value: String) {
        analytics.setUserProperty(key, value)
    }
}
```

---

## Compose画面追跡

```kotlin
@Composable
fun TrackScreenView(screenName: String, analyticsHelper: AnalyticsHelper) {
    DisposableEffect(screenName) {
        analyticsHelper.logScreenView(screenName)
        onDispose { }
    }
}

// Navigation連携で自動追跡
@Composable
fun AnalyticsNavHost(analyticsHelper: AnalyticsHelper) {
    val navController = rememberNavController()

    LaunchedEffect(navController) {
        navController.currentBackStackEntryFlow.collect { entry ->
            val route = entry.destination.route ?: return@collect
            analyticsHelper.logScreenView(route)
        }
    }

    NavHost(navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onItemClick = { id ->
                analyticsHelper.logEvent("item_click", mapOf("item_id" to id))
                navController.navigate("detail/$id")
            })
        }
        composable("detail/{id}") { DetailScreen() }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `logEvent` | カスタムイベント |
| `SCREEN_VIEW` | 画面表示追跡 |
| `setUserProperty` | ユーザー属性 |
| `currentBackStackEntryFlow` | 自動画面追跡 |

- `Firebase.analytics.logEvent()`でイベント計測
- `DisposableEffect`で画面表示をトリガー
- `currentBackStackEntryFlow`でNavigation遷移を自動追跡
- ユーザープロパティでセグメント分析

---

8種類のAndroidアプリテンプレート（Firebase対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FirebaseCrashlytics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-crashlytics-2026)
- [Compose FirebaseAuth](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-auth-2026)
- [Compose FirebaseFirestore](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-firestore-2026)
