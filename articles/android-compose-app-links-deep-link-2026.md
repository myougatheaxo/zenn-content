---
title: "App Links / Deep Link完全ガイド — URL→アプリ遷移"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "deeplink"]
published: true
---

## この記事で学べること

**App Links / Deep Link**（URLからアプリ起動、Navigation Deep Link、ドメイン検証、DAL設定）を解説します。

---

## AndroidManifest設定

```xml
<activity android:name=".MainActivity"
    android:exported="true">

    <!-- App Links（検証済みドメイン） -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https"
              android:host="example.com"
              android:pathPrefix="/item" />
    </intent-filter>

    <!-- Custom URL Scheme -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="myapp"
              android:host="detail" />
    </intent-filter>
</activity>
```

---

## Digital Asset Links（DAL）

```json
// https://example.com/.well-known/assetlinks.json
[{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
        "namespace": "android_app",
        "package_name": "com.example.myapp",
        "sha256_cert_fingerprints": [
            "AA:BB:CC:DD:..."
        ]
    }
}]
```

```bash
# SHA256フィンガープリント取得
keytool -list -v -keystore release.keystore \
  -alias mykey | grep SHA256
```

---

## Navigation Deep Link

```kotlin
NavHost(navController, startDestination = "home") {
    composable("home") { HomeScreen() }

    composable(
        route = "item/{itemId}",
        arguments = listOf(navArgument("itemId") { type = NavType.StringType }),
        deepLinks = listOf(
            navDeepLink {
                uriPattern = "https://example.com/item/{itemId}"
            },
            navDeepLink {
                uriPattern = "myapp://detail/{itemId}"
            }
        )
    ) { backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
        ItemDetailScreen(itemId)
    }

    composable(
        route = "search",
        deepLinks = listOf(
            navDeepLink {
                uriPattern = "https://example.com/search?q={query}"
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
}
```

---

## Intent処理

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            val navController = rememberNavController()

            // Deep Linkの処理
            LaunchedEffect(Unit) {
                handleDeepLink(intent, navController)
            }

            AppNavHost(navController)
        }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        // アプリが既に起動中の場合
        setIntent(intent)
    }

    private fun handleDeepLink(intent: Intent, navController: NavController) {
        intent.data?.let { uri ->
            when {
                uri.pathSegments.firstOrNull() == "item" -> {
                    val itemId = uri.lastPathSegment
                    navController.navigate("item/$itemId")
                }
                uri.path == "/search" -> {
                    val query = uri.getQueryParameter("q") ?: ""
                    navController.navigate("search?query=$query")
                }
            }
        }
    }
}
```

---

## テスト

```bash
# Deep Linkテスト
adb shell am start -W -a android.intent.action.VIEW \
  -d "https://example.com/item/123" com.example.myapp

# Custom Schemeテスト
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://detail/456"

# App Links検証確認
adb shell pm get-app-links com.example.myapp
```

---

## まとめ

| 種類 | スキーム | 検証 |
|------|---------|------|
| App Links | `https://` | DAL必須 |
| Deep Link | `https://` | 不要（選択ダイアログ） |
| Custom Scheme | `myapp://` | 不要 |

- App Linksは`autoVerify=true`+DALで直接起動
- Navigation Composeの`deepLinks`で自動処理
- `onNewIntent`でアプリ起動中のリンク処理
- adbでDeep Linkテスト可能

---

8種類のAndroidアプリテンプレート（Deep Link対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [Share/Intent](https://zenn.dev/myougatheaxo/articles/android-compose-share-intent-2026)
- [Firebase Dynamic Links代替](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
