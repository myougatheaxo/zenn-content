---
title: "Compose Deep Link完全ガイド — NavDeepLink/URI引数/Pending Intent連携"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "navigation"]
published: true
---

## この記事で学べること

**Compose Deep Link**（NavDeepLink定義、URI引数、通知からのDeep Link、テスト方法）を解説します。

---

## NavDeepLink定義

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "home") {
        composable("home") { HomeScreen(navController) }

        composable(
            route = "item/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.IntType }),
            deepLinks = listOf(
                navDeepLink {
                    uriPattern = "https://example.com/item/{itemId}"
                    action = Intent.ACTION_VIEW
                },
                navDeepLink {
                    uriPattern = "myapp://item/{itemId}"
                }
            )
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getInt("itemId") ?: 0
            ItemDetailScreen(itemId)
        }

        composable(
            route = "search?query={query}",
            arguments = listOf(navArgument("query") { type = NavType.StringType; defaultValue = "" }),
            deepLinks = listOf(
                navDeepLink { uriPattern = "myapp://search?query={query}" }
            )
        ) { backStackEntry ->
            val query = backStackEntry.arguments?.getString("query") ?: ""
            SearchScreen(initialQuery = query)
        }
    }
}
```

---

## Manifest宣言

```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" />
        <data android:scheme="myapp" />
    </intent-filter>
</activity>
```

---

## 通知からのDeep Link

```kotlin
fun createDeepLinkNotification(context: Context, itemId: Int) {
    val deepLinkIntent = Intent(
        Intent.ACTION_VIEW,
        "myapp://item/$itemId".toUri(),
        context,
        MainActivity::class.java
    )

    val pendingIntent = TaskStackBuilder.create(context).run {
        addNextIntentWithParentStack(deepLinkIntent)
        getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
    }

    val notification = NotificationCompat.Builder(context, "items")
        .setContentTitle("新しいアイテム")
        .setContentText("アイテム #$itemId が追加されました")
        .setSmallIcon(R.drawable.ic_notification)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)
        .build()

    NotificationManagerCompat.from(context).notify(itemId, notification)
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| Deep Link定義 | `navDeepLink { uriPattern }` |
| カスタムスキーム | `myapp://path` |
| HTTPリンク | `https://domain/path` |
| 通知連携 | `TaskStackBuilder` |

- `navDeepLink`でNavigation Compose内にDeep Link定義
- カスタムスキーム+HTTPSリンク両対応
- `TaskStackBuilder`で通知からのバックスタック構築
- `adb shell am start`でDeep Linkテスト

---

8種類のAndroidアプリテンプレート（Deep Link対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [App Links](https://zenn.dev/myougatheaxo/articles/android-compose-app-links-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-local-notification-2026)
