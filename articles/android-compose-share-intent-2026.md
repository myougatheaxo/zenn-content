---
title: "共有/インテント連携ガイド — ShareSheet/DeepLink/カスタムURL"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "intent"]
published: true
---

## この記事で学べること

Androidの**共有/インテント連携**（ShareSheet、テキスト/画像共有、Deep Link、カスタムURLスキーム）を解説します。

---

## テキスト共有

```kotlin
@Composable
fun ShareTextButton(text: String) {
    val context = LocalContext.current

    Button(onClick = {
        val sendIntent = Intent().apply {
            action = Intent.ACTION_SEND
            putExtra(Intent.EXTRA_TEXT, text)
            type = "text/plain"
        }
        val shareIntent = Intent.createChooser(sendIntent, "共有先を選択")
        context.startActivity(shareIntent)
    }) {
        Icon(Icons.Default.Share, null)
        Spacer(Modifier.width(8.dp))
        Text("共有")
    }
}

// URL + タイトル共有
fun shareUrl(context: Context, url: String, title: String) {
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "text/plain"
        putExtra(Intent.EXTRA_SUBJECT, title)
        putExtra(Intent.EXTRA_TEXT, "$title\n$url")
    }
    context.startActivity(Intent.createChooser(intent, null))
}
```

---

## 画像共有

```kotlin
fun shareImage(context: Context, imageUri: Uri, text: String? = null) {
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "image/*"
        putExtra(Intent.EXTRA_STREAM, imageUri)
        text?.let { putExtra(Intent.EXTRA_TEXT, it) }
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    context.startActivity(Intent.createChooser(intent, "画像を共有"))
}

// Bitmap→Uri変換して共有
suspend fun shareBitmap(context: Context, bitmap: Bitmap) {
    withContext(Dispatchers.IO) {
        val file = File(context.cacheDir, "share_${System.currentTimeMillis()}.png")
        file.outputStream().use { bitmap.compress(Bitmap.CompressFormat.PNG, 100, it) }

        val uri = FileProvider.getUriForFile(
            context, "${context.packageName}.fileprovider", file
        )

        withContext(Dispatchers.Main) {
            shareImage(context, uri)
        }
    }
}
```

---

## 共有を受け取る

```kotlin
// AndroidManifest.xml
// <activity android:name=".MainActivity">
//     <intent-filter>
//         <action android:name="android.intent.action.SEND" />
//         <category android:name="android.intent.category.DEFAULT" />
//         <data android:mimeType="text/plain" />
//     </intent-filter>
// </activity>

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val sharedText = when (intent?.action) {
            Intent.ACTION_SEND -> intent.getStringExtra(Intent.EXTRA_TEXT)
            else -> null
        }

        setContent {
            AppTheme {
                if (sharedText != null) {
                    SharedContentScreen(text = sharedText)
                } else {
                    MainScreen()
                }
            }
        }
    }
}
```

---

## Deep Link

```kotlin
// AndroidManifest.xml
// <intent-filter android:autoVerify="true">
//     <action android:name="android.intent.action.VIEW" />
//     <category android:name="android.intent.category.DEFAULT" />
//     <category android:name="android.intent.category.BROWSABLE" />
//     <data android:scheme="https"
//           android:host="myapp.example.com"
//           android:pathPrefix="/item/" />
// </intent-filter>

// Navigation Compose
NavHost(navController, startDestination = "home") {
    composable(
        "item/{id}",
        deepLinks = listOf(
            navDeepLink {
                uriPattern = "https://myapp.example.com/item/{id}"
            }
        ),
        arguments = listOf(
            navArgument("id") { type = NavType.StringType }
        )
    ) { backStackEntry ->
        val id = backStackEntry.arguments?.getString("id") ?: ""
        ItemDetailScreen(id = id)
    }
}
```

---

## カスタムURLスキーム

```kotlin
// AndroidManifest.xml
// <intent-filter>
//     <action android:name="android.intent.action.VIEW" />
//     <category android:name="android.intent.category.DEFAULT" />
//     <category android:name="android.intent.category.BROWSABLE" />
//     <data android:scheme="myapp" android:host="action" />
// </intent-filter>

// myapp://action/open?id=123

// 他アプリからの起動
fun openWithCustomScheme(context: Context, itemId: String) {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse("myapp://action/open?id=$itemId"))
    context.startActivity(intent)
}
```

---

## 外部アプリ連携

```kotlin
// メール送信
fun sendEmail(context: Context, to: String, subject: String, body: String) {
    val intent = Intent(Intent.ACTION_SENDTO).apply {
        data = Uri.parse("mailto:")
        putExtra(Intent.EXTRA_EMAIL, arrayOf(to))
        putExtra(Intent.EXTRA_SUBJECT, subject)
        putExtra(Intent.EXTRA_TEXT, body)
    }
    if (intent.resolveActivity(context.packageManager) != null) {
        context.startActivity(intent)
    }
}

// 電話発信
fun dialPhone(context: Context, number: String) {
    val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:$number"))
    context.startActivity(intent)
}

// ブラウザで開く
fun openBrowser(context: Context, url: String) {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
    context.startActivity(intent)
}

// Google Mapsで開く
fun openMaps(context: Context, lat: Double, lng: Double, label: String) {
    val uri = Uri.parse("geo:$lat,$lng?q=$lat,$lng($label)")
    val intent = Intent(Intent.ACTION_VIEW, uri).apply {
        setPackage("com.google.android.apps.maps")
    }
    context.startActivity(intent)
}
```

---

## まとめ

| 機能 | Intent Action |
|------|--------------|
| テキスト共有 | `ACTION_SEND` + `text/plain` |
| 画像共有 | `ACTION_SEND` + `image/*` |
| Deep Link | `ACTION_VIEW` + URI |
| メール | `ACTION_SENDTO` + `mailto:` |
| 電話 | `ACTION_DIAL` + `tel:` |
| ブラウザ | `ACTION_VIEW` + URL |

- `Intent.createChooser`でShareSheet表示
- `FileProvider`で画像URI共有（FLAG_GRANT_READ_URI_PERMISSION必須）
- `navDeepLink`でNavigation ComposのDeep Link対応
- `autoVerify="true"`でApp Links（HTTPS Deep Link）

---

8種類のAndroidアプリテンプレート（インテント連携実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DeepLink Handler](https://zenn.dev/myougatheaxo/articles/android-compose-deeplink-handler-2026)
- [Navigation引数](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-args-2026)
- [画像ピッカー](https://zenn.dev/myougatheaxo/articles/android-compose-image-picker-camera-2026)
