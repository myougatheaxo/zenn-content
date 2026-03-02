---
title: "App Links完全ガイド — ディープリンク/検証/Navigation連携"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "deeplink"]
published: true
---

## この記事で学べること

**App Links**（Android App Links、ディープリンク検証、Intent Filter、Navigation連携）を解説します。

---

## AndroidManifest設定

```xml
<activity android:name=".MainActivity"
    android:exported="true">

    <!-- App Links (HTTPS、自動検証) -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
            android:host="example.com"
            android:pathPrefix="/product/" />
    </intent-filter>

    <!-- カスタムスキーム (ディープリンク) -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp"
            android:host="product" />
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
            "AA:BB:CC:DD:..."
        ]
    }
}]
```

```bash
# SHA256フィンガープリント取得
keytool -list -v -keystore release.keystore | grep SHA256
```

---

## Navigation連携

```kotlin
@Composable
fun AppNavHost(navController: NavHostController) {
    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> { HomeScreen() }

        composable<ProductRoute>(
            deepLinks = listOf(
                navDeepLink<ProductRoute>(basePath = "https://example.com/product"),
                navDeepLink<ProductRoute>(basePath = "myapp://product")
            )
        ) { backStackEntry ->
            val route = backStackEntry.toRoute<ProductRoute>()
            ProductScreen(productId = route.id)
        }
    }
}

@Serializable
data class ProductRoute(val id: String)
```

---

## Intent処理

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val navController = rememberNavController()
            AppNavHost(navController)

            // ディープリンク処理
            LaunchedEffect(Unit) {
                handleDeepLink(intent, navController)
            }
        }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        // アプリ起動中にリンクを受信
    }

    private fun handleDeepLink(intent: Intent, navController: NavController) {
        intent.data?.let { uri ->
            when {
                uri.pathSegments.firstOrNull() == "product" -> {
                    val productId = uri.lastPathSegment ?: return
                    navController.navigate(ProductRoute(productId))
                }
            }
        }
    }
}
```

---

## テスト

```bash
# App Linksテスト
adb shell am start -W -a android.intent.action.VIEW \
    -d "https://example.com/product/123" com.example.app

# カスタムスキームテスト
adb shell am start -W -a android.intent.action.VIEW \
    -d "myapp://product/123"

# App Links検証状態確認
adb shell pm get-app-links com.example.app
```

---

## まとめ

| 種類 | スキーム | 検証 |
|------|---------|------|
| App Links | `https://` | 自動検証 |
| Deep Links | `myapp://` | 検証なし |
| Web Links | `http(s)://` | ダイアログ表示 |

- `autoVerify="true"`でApp Linksを自動検証
- `assetlinks.json`をサーバーに配置
- Navigation Composeの`deepLinks`で宣言的にルート設定
- `adb`コマンドでディープリンクをテスト

---

8種類のAndroidアプリテンプレート（ディープリンク対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [Share Intent](https://zenn.dev/myougatheaxo/articles/android-compose-share-intent-2026)
- [通知](https://zenn.dev/myougatheaxo/articles/android-compose-notification-advanced-2026)
