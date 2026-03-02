---
title: "Compose DeepLink完全ガイド — ディープリンク/URI処理/通知連携"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**Compose DeepLink**（ディープリンク設定、URI処理、通知からの遷移、NavDeepLink）を解説します。

---

## 基本DeepLink

```kotlin
// AndroidManifest.xml
// <activity>
//   <intent-filter>
//     <action android:name="android.intent.action.VIEW" />
//     <category android:name="android.intent.category.DEFAULT" />
//     <category android:name="android.intent.category.BROWSABLE" />
//     <data android:scheme="myapp" android:host="detail" />
//   </intent-filter>
// </activity>

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "home") {
        composable("home") { HomeScreen(navController) }
        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.LongType }),
            deepLinks = listOf(
                navDeepLink { uriPattern = "myapp://detail/{itemId}" },
                navDeepLink { uriPattern = "https://example.com/items/{itemId}" }
            )
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getLong("itemId") ?: 0L
            DetailScreen(itemId)
        }
    }
}
```

---

## 通知からのDeepLink

```kotlin
fun createNotificationDeepLink(context: Context, itemId: Long) {
    val deepLinkIntent = Intent(
        Intent.ACTION_VIEW,
        "myapp://detail/$itemId".toUri(),
        context,
        MainActivity::class.java
    )

    val pendingIntent = TaskStackBuilder.create(context).run {
        addNextIntentWithParentStack(deepLinkIntent)
        getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
    }

    val notification = NotificationCompat.Builder(context, "channel_id")
        .setSmallIcon(R.drawable.ic_notification)
        .setContentTitle("新しい通知")
        .setContentText("アイテム #$itemId が更新されました")
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .build()

    NotificationManagerCompat.from(context).notify(itemId.toInt(), notification)
}
```

---

## クエリパラメータ

```kotlin
composable(
    route = "search",
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "myapp://search?query={query}"
        }
    ),
    arguments = listOf(
        navArgument("query") {
            type = NavType.StringType
            defaultValue = ""
        }
    )
) { backStackEntry ->
    val query = backStackEntry.arguments?.getString("query") ?: ""
    SearchScreen(initialQuery = query)
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `navDeepLink` | ディープリンク定義 |
| `uriPattern` | URIパターン |
| `TaskStackBuilder` | バックスタック構築 |
| `PendingIntent` | 通知連携 |

- `navDeepLink`でURIパターンを定義
- カスタムスキーム(`myapp://`)とHTTPSスキームの両方対応
- `TaskStackBuilder`で適切なバックスタックを構築
- AndroidManifestにintent-filter設定が必要

---

8種類のAndroidアプリテンプレート（ナビゲーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation TypeSafe](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-type-safe-2026)
- [Compose Notification](https://zenn.dev/myougatheaxo/articles/android-compose-compose-notification-2026)
- [Navigation Result](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-result-2026)
