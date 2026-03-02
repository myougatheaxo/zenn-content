---
title: "クリップボード・共有機能ガイド — Composeでコピー&シェアを実装"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "share"]
published: true
---

## この記事で学べること

Composeアプリでの**クリップボードコピー**と**共有（Share）機能**の実装方法を解説します。

---

## クリップボードにコピー

```kotlin
@Composable
fun CopyToClipboard() {
    val clipboardManager = LocalClipboardManager.current

    var text by remember { mutableStateOf("コピーするテキスト") }

    Row(verticalAlignment = Alignment.CenterVertically) {
        Text(text, modifier = Modifier.weight(1f))
        IconButton(onClick = {
            clipboardManager.setText(AnnotatedString(text))
        }) {
            Icon(Icons.Default.ContentCopy, "コピー")
        }
    }
}
```

---

## コピー完了のSnackbar

```kotlin
@Composable
fun CopyWithFeedback() {
    val clipboardManager = LocalClipboardManager.current
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(Modifier.padding(padding)) {
            Button(onClick = {
                clipboardManager.setText(AnnotatedString("ABC-123"))
                scope.launch {
                    snackbarHostState.showSnackbar("コピーしました")
                }
            }) {
                Text("コードをコピー")
            }
        }
    }
}
```

---

## クリップボードから貼り付け

```kotlin
@Composable
fun PasteFromClipboard() {
    val clipboardManager = LocalClipboardManager.current
    var text by remember { mutableStateOf("") }

    Row {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier.weight(1f)
        )
        IconButton(onClick = {
            clipboardManager.getText()?.let {
                text = it.text
            }
        }) {
            Icon(Icons.Default.ContentPaste, "貼り付け")
        }
    }
}
```

---

## テキスト共有

```kotlin
@Composable
fun ShareText() {
    val context = LocalContext.current

    Button(onClick = {
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_TEXT, "この記事をチェック！ https://example.com")
            putExtra(Intent.EXTRA_SUBJECT, "おすすめ記事")
        }
        context.startActivity(Intent.createChooser(intent, "共有"))
    }) {
        Icon(Icons.Default.Share, null)
        Spacer(Modifier.width(8.dp))
        Text("共有")
    }
}
```

---

## 画像共有

```kotlin
fun shareImage(context: Context, uri: Uri) {
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "image/*"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    context.startActivity(Intent.createChooser(intent, "画像を共有"))
}
```

---

## 共有の受信

```kotlin
// AndroidManifest.xml
<activity android:name=".ShareReceiveActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
</activity>

// Activity
class ShareReceiveActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val sharedText = intent.getStringExtra(Intent.EXTRA_TEXT) ?: ""
        setContent {
            ReceivedContent(sharedText)
        }
    }
}
```

---

## まとめ

- `LocalClipboardManager`でコピー/貼り付け
- `Intent.ACTION_SEND`でテキスト・画像共有
- `Intent.createChooser`で共有先選択
- `intent-filter`で共有の受信
- Snackbarでコピー完了フィードバック

---

8種類のAndroidアプリテンプレート（共有機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Snackbar完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-snackbar-2026)
- [Deep Link実装ガイド](https://zenn.dev/myougatheaxo/articles/android-deeplink-guide-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
