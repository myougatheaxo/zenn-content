---
title: "Compose ShareIntent完全ガイド — 共有/テキスト/画像/ファイル共有"
emoji: "📤"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "intent"]
published: true
---

## この記事で学べること

**Compose ShareIntent**（テキスト共有、画像共有、ファイル共有、ShareSheet制御）を解説します。

---

## テキスト共有

```kotlin
@Composable
fun ShareTextButton(text: String) {
    val context = LocalContext.current

    Button(onClick = {
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_TEXT, text)
            putExtra(Intent.EXTRA_SUBJECT, "共有タイトル")
        }
        context.startActivity(Intent.createChooser(intent, "共有先を選択"))
    }) {
        Icon(Icons.Default.Share, null)
        Spacer(Modifier.width(8.dp))
        Text("共有")
    }
}
```

---

## 画像共有（FileProvider）

```kotlin
@Composable
fun ShareImageButton(imageFile: File) {
    val context = LocalContext.current

    IconButton(onClick = {
        val uri = FileProvider.getUriForFile(
            context, "${context.packageName}.fileprovider", imageFile
        )

        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "image/*"
            putExtra(Intent.EXTRA_STREAM, uri)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(Intent.createChooser(intent, "画像を共有"))
    }) {
        Icon(Icons.Default.Share, "共有")
    }
}

// AndroidManifest.xml
// <provider
//     android:name="androidx.core.content.FileProvider"
//     android:authorities="${applicationId}.fileprovider"
//     android:exported="false"
//     android:grantUriPermissions="true">
//     <meta-data
//         android:name="android.support.FILE_PROVIDER_PATHS"
//         android:resource="@xml/file_paths" />
// </provider>
```

---

## 共有受信

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val sharedText = remember {
                if (intent?.action == Intent.ACTION_SEND && intent.type == "text/plain") {
                    intent.getStringExtra(Intent.EXTRA_TEXT)
                } else null
            }

            if (sharedText != null) {
                SharedContentScreen(sharedText)
            } else {
                MainScreen()
            }
        }
    }
}

@Composable
fun SharedContentScreen(text: String) {
    Column(Modifier.padding(16.dp)) {
        Text("受信した共有テキスト:", style = MaterialTheme.typography.titleMedium)
        Card(Modifier.fillMaxWidth().padding(top = 8.dp)) {
            Text(text, Modifier.padding(16.dp))
        }
    }
}
```

---

## まとめ

| 方法 | 用途 |
|------|------|
| `ACTION_SEND` | 単一コンテンツ共有 |
| `ACTION_SEND_MULTIPLE` | 複数コンテンツ |
| `FileProvider` | ファイルURI生成 |
| `createChooser` | 共有先選択UI |

- テキスト共有は`EXTRA_TEXT`で設定
- ファイル共有は`FileProvider`でURIを生成
- `FLAG_GRANT_READ_URI_PERMISSION`で読み取り許可
- 共有受信はintent-filterとActivity内で処理

---

8種類のAndroidアプリテンプレート（共有機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FilePicker](https://zenn.dev/myougatheaxo/articles/android-compose-compose-file-picker-2026)
- [Compose Clipboard](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clipboard-2026)
- [Compose DeepLink](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-deep-link-2026)
