---
title: "Androidディープリンク完全ガイド — URLからアプリの特定画面を開く"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "deeplink"]
published: true
---

## この記事で学べること

ディープリンクを使えば、URLやQRコードから**アプリの特定画面を直接開く**ことができます。Compose Navigationでの実装方法を解説します。

---

## ディープリンクの種類

| 種類 | 特徴 |
|------|------|
| 標準ディープリンク | `myapp://detail/123` カスタムスキーム |
| App Links (Android) | `https://example.com/detail/123` HTTPSスキーム、検証済み |
| Web URL | ブラウザでも開ける |

**App Links**が推奨。HTTPSなのでセキュアで、他のアプリに乗っ取られません。

---

## Compose Navigationでのディープリンク

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "home") {
        composable("home") {
            HomeScreen(navController)
        }

        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.IntType }),
            deepLinks = listOf(
                navDeepLink {
                    uriPattern = "https://example.com/detail/{itemId}"
                }
            )
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getInt("itemId") ?: 0
            DetailScreen(itemId)
        }
    }
}
```

---

## AndroidManifest.xml

```xml
<activity
    android:name=".MainActivity"
    android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="example.com"
            android:pathPrefix="/detail" />
    </intent-filter>
</activity>
```

`android:autoVerify="true"`でApp Linksの自動検証が有効になります。

---

## カスタムスキーム

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data
        android:scheme="myapp"
        android:host="open" />
</intent-filter>
```

```kotlin
composable(
    route = "profile/{userId}",
    deepLinks = listOf(
        navDeepLink { uriPattern = "myapp://open/profile/{userId}" }
    )
) { backStackEntry ->
    val userId = backStackEntry.arguments?.getString("userId") ?: ""
    ProfileScreen(userId)
}
```

`myapp://open/profile/user123`でプロフィール画面が開きます。

---

## ディープリンクのテスト

```bash
# adbコマンドでテスト
adb shell am start -a android.intent.action.VIEW \
    -d "https://example.com/detail/42" \
    com.example.myapp
```

---

## 複数パラメータ

```kotlin
composable(
    route = "search?query={query}&category={category}",
    arguments = listOf(
        navArgument("query") { defaultValue = "" },
        navArgument("category") { defaultValue = "all" }
    ),
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "https://example.com/search?query={query}&category={category}"
        }
    )
) { backStackEntry ->
    val query = backStackEntry.arguments?.getString("query") ?: ""
    val category = backStackEntry.arguments?.getString("category") ?: "all"
    SearchScreen(query, category)
}
```

---

## まとめ

- **App Links**（HTTPS）が推奨 — セキュアで検証可能
- Compose Navigationの`navDeepLink`で宣言的に定義
- ManifestにIntent Filterを追加
- `adb shell am start`でテスト
- クエリパラメータも対応可能

---

8種類のAndroidアプリテンプレート（ナビゲーション設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
