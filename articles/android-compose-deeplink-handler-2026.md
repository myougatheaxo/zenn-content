---
title: "ディープリンク実装ガイド — Navigation Compose + App Links"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "deeplink"]
published: true
---

## この記事で学べること

Navigation Composeでの**ディープリンク/App Links**の実装を解説します。

---

## AndroidManifest設定

```xml
<activity android:name=".MainActivity"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

    <!-- ディープリンク -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp" android:host="item" />
    </intent-filter>

    <!-- App Links (HTTPS) -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" android:pathPrefix="/item" />
    </intent-filter>
</activity>
```

---

## Navigation Composeでのディープリンク

```kotlin
@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen()
        }

        composable(
            "item/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType }),
            deepLinks = listOf(
                navDeepLink { uriPattern = "myapp://item/{itemId}" },
                navDeepLink { uriPattern = "https://example.com/item/{itemId}" }
            )
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId") ?: return@composable
            ItemDetailScreen(itemId)
        }

        composable(
            "profile/{userId}",
            deepLinks = listOf(
                navDeepLink { uriPattern = "myapp://profile/{userId}" }
            )
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getString("userId") ?: return@composable
            ProfileScreen(userId)
        }
    }
}
```

---

## 型安全ルートとディープリンク

```kotlin
@Serializable
data class ItemRoute(val itemId: String)

NavHost(navController = navController, startDestination = HomeRoute) {
    composable<ItemRoute>(
        deepLinks = listOf(
            navDeepLink<ItemRoute>(basePath = "https://example.com/item")
        )
    ) { backStackEntry ->
        val route: ItemRoute = backStackEntry.toRoute()
        ItemDetailScreen(route.itemId)
    }
}
```

---

## アプリ内からのディープリンク遷移

```kotlin
@Composable
fun ShareableContent(itemId: String) {
    val context = LocalContext.current

    Button(onClick = {
        val deepLink = "myapp://item/$itemId"
        // クリップボードにコピー
        val clipboard = context.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
        clipboard.setPrimaryClip(ClipData.newPlainText("link", deepLink))
    }) {
        Text("リンクをコピー")
    }
}

// テスト用: adbコマンド
// adb shell am start -W -a android.intent.action.VIEW -d "myapp://item/123"
// adb shell am start -W -a android.intent.action.VIEW -d "https://example.com/item/123"
```

---

## App Links検証

```json
// .well-known/assetlinks.json (サーバー側に配置)
[{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
        "namespace": "android_app",
        "package_name": "com.example.myapp",
        "sha256_cert_fingerprints": [
            "AA:BB:CC:..."
        ]
    }
}]
```

```bash
# 署名のフィンガープリント取得
keytool -list -v -keystore release.keystore | grep SHA256
```

---

## まとめ

- `navDeepLink { uriPattern = "..." }`でディープリンク定義
- カスタムスキーム(`myapp://`)とHTTPS両方対応
- 型安全ルートは`navDeepLink<Route>(basePath = ...)`
- App Linksは`assetlinks.json`で検証
- `android:autoVerify="true"`で自動検証
- adbコマンドでテスト可能

---

8種類のAndroidアプリテンプレート（ディープリンク対応設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ディープリンク完全ガイド](https://zenn.dev/myougatheaxo/articles/android-deeplink-guide-2026)
- [型安全Navigationガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-typesafe-2026)
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
