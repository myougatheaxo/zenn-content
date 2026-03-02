---
title: "Intent/共有ガイド — 外部アプリ連携/暗黙的Intent/共有シート"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "intent"]
published: true
---

## この記事で学べること

Composeでの**Intent連携**（外部アプリ起動、共有シート、ファイル共有）を解説します。

---

## 外部アプリ起動

```kotlin
@Composable
fun ExternalAppLaunchers() {
    val context = LocalContext.current

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // ブラウザ
        Button(onClick = {
            val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))
            context.startActivity(intent)
        }) { Text("ブラウザで開く") }

        // 電話
        Button(onClick = {
            val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:09012345678"))
            context.startActivity(intent)
        }) { Text("電話をかける") }

        // メール
        Button(onClick = {
            val intent = Intent(Intent.ACTION_SENDTO).apply {
                data = Uri.parse("mailto:support@example.com")
                putExtra(Intent.EXTRA_SUBJECT, "お問い合わせ")
            }
            context.startActivity(intent)
        }) { Text("メールを送る") }

        // マップ
        Button(onClick = {
            val intent = Intent(Intent.ACTION_VIEW, Uri.parse("geo:35.6812,139.7671?q=東京駅"))
            context.startActivity(intent)
        }) { Text("地図で開く") }

        // 設定画面
        Button(onClick = {
            val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                data = Uri.fromParts("package", context.packageName, null)
            }
            context.startActivity(intent)
        }) { Text("アプリ設定") }
    }
}
```

---

## テキスト共有

```kotlin
@Composable
fun ShareText(title: String, content: String) {
    val context = LocalContext.current

    IconButton(onClick = {
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_SUBJECT, title)
            putExtra(Intent.EXTRA_TEXT, content)
        }
        context.startActivity(Intent.createChooser(intent, "共有"))
    }) {
        Icon(Icons.Default.Share, "共有")
    }
}
```

---

## ファイル共有（FileProvider）

```kotlin
// 画像を共有
fun shareImage(context: Context, imageFile: File) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        imageFile
    )
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "image/*"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    context.startActivity(Intent.createChooser(intent, "画像を共有"))
}
```

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>

<!-- res/xml/file_paths.xml -->
<paths>
    <cache-path name="images" path="images/" />
    <files-path name="documents" path="documents/" />
</paths>
```

---

## Intentの受信

```kotlin
// MainActivityで受け取り
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val sharedText = remember {
                when (intent?.action) {
                    Intent.ACTION_SEND -> intent.getStringExtra(Intent.EXTRA_TEXT)
                    else -> null
                }
            }
            AppContent(initialSharedText = sharedText)
        }
    }
}
```

---

## まとめ

- `Intent.ACTION_VIEW`でブラウザ/マップ/電話
- `Intent.ACTION_SEND`でテキスト/ファイル共有
- `Intent.createChooser()`で共有シート表示
- `FileProvider`でファイルを安全に共有
- `Intent.ACTION_SENDTO`でメール送信
- `Settings.ACTION_APPLICATION_DETAILS_SETTINGS`でアプリ設定

---

8種類のAndroidアプリテンプレート（外部連携設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [クリップボード/共有ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-clipboard-text-2026)
- [ディープリンクガイド](https://zenn.dev/myougatheaxo/articles/android-compose-deeplink-handler-2026)
- [ファイルストレージガイド](https://zenn.dev/myougatheaxo/articles/android-file-storage-2026)
