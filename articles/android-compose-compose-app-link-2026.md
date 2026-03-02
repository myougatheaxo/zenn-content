---
title: "Compose AppLink完全ガイド — App Links/ディープリンク検証/Digital Asset Links"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "deeplink"]
published: true
---

## この記事で学べること

**Compose AppLink**（App Links、Digital Asset Links、ディープリンク検証、intent-filter設定）を解説します。

---

## AndroidManifest設定

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity" android:exported="true">
    <!-- App Links（自動検証） -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com"
              android:pathPrefix="/product" />
    </intent-filter>
</activity>
```

---

## Digital Asset Links

```json
// https://example.com/.well-known/assetlinks.json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": [
      "AB:CD:EF:12:34:56:..."
    ]
  }
}]
```

```kotlin
// SHA256フィンガープリント取得
// keytool -list -v -keystore your-keystore.jks
// または
// ./gradlew signingReport
```

---

## Compose内でのディープリンク処理

```kotlin
// NavGraph設定
@Composable
fun AppNavHost() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "home") {
        composable("home") { HomeScreen(navController) }

        composable(
            route = "product/{id}",
            deepLinks = listOf(
                navDeepLink { uriPattern = "https://example.com/product/{id}" }
            ),
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id") ?: ""
            ProductDetailScreen(id)
        }

        composable(
            route = "search",
            deepLinks = listOf(
                navDeepLink { uriPattern = "https://example.com/search?q={query}" }
            ),
            arguments = listOf(navArgument("query") { defaultValue = "" })
        ) { backStackEntry ->
            val query = backStackEntry.arguments?.getString("query") ?: ""
            SearchScreen(query)
        }
    }
}

@Composable
fun ProductDetailScreen(productId: String) {
    Column(Modifier.padding(16.dp)) {
        Text("商品ID: $productId", style = MaterialTheme.typography.headlineMedium)
        Text("App Linkから開かれました", style = MaterialTheme.typography.bodyMedium)
    }
}
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `autoVerify="true"` | App Links自動検証 |
| `assetlinks.json` | ドメイン所有証明 |
| `navDeepLink` | Compose内ディープリンク |
| `sha256_cert_fingerprints` | アプリ署名検証 |

- `android:autoVerify="true"`でApp Links有効化
- `.well-known/assetlinks.json`をWebサーバーに配置
- `navDeepLink`でCompose Navigationと統合
- `signingReport`でSHA256フィンガープリント取得

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-deep-link-2026)
- [Compose Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-2026)
- [Compose ShareIntent](https://zenn.dev/myougatheaxo/articles/android-compose-compose-share-intent-2026)
