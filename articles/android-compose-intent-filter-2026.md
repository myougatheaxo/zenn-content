---
title: "Intent/IntentFilter完全ガイド — 暗黙的Intent/データ共有/カスタムスキーム"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "intent"]
published: true
---

## この記事で学べること

**Intent/IntentFilter**（暗黙的Intent、データ共有、カスタムURLスキーム、ActivityResultContract）を解説します。

---

## 暗黙的Intent

```kotlin
@Composable
fun IntentExamples() {
    val context = LocalContext.current

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        // ブラウザで開く
        Button(onClick = {
            val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://example.com"))
            context.startActivity(intent)
        }) { Text("ブラウザで開く") }

        // メール送信
        Button(onClick = {
            val intent = Intent(Intent.ACTION_SENDTO).apply {
                data = Uri.parse("mailto:")
                putExtra(Intent.EXTRA_EMAIL, arrayOf("test@example.com"))
                putExtra(Intent.EXTRA_SUBJECT, "件名")
            }
            context.startActivity(Intent.createChooser(intent, "メールアプリ"))
        }) { Text("メール送信") }

        // 共有
        Button(onClick = {
            val intent = Intent(Intent.ACTION_SEND).apply {
                type = "text/plain"
                putExtra(Intent.EXTRA_TEXT, "共有テキスト")
            }
            context.startActivity(Intent.createChooser(intent, "共有"))
        }) { Text("テキスト共有") }
    }
}
```

---

## データ受信（IntentFilter）

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".ShareReceiverActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="image/*" />
    </intent-filter>
</activity>
```

```kotlin
class ShareReceiverActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            when (intent?.action) {
                Intent.ACTION_SEND -> {
                    val text = intent.getStringExtra(Intent.EXTRA_TEXT)
                    val imageUri = intent.getParcelableExtra<Uri>(Intent.EXTRA_STREAM)
                    SharedContentScreen(text = text, imageUri = imageUri)
                }
            }
        }
    }
}
```

---

## ActivityResultContract

```kotlin
@Composable
fun FilePickerExample() {
    var selectedUri by remember { mutableStateOf<Uri?>(null) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocument()
    ) { uri -> selectedUri = uri }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { launcher.launch(arrayOf("application/pdf")) }) {
            Text("PDFを選択")
        }
        selectedUri?.let {
            Text("選択: ${it.lastPathSegment}")
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 暗黙Intent | `Intent(ACTION_VIEW)` |
| データ共有 | `ACTION_SEND` + Chooser |
| 受信 | `IntentFilter` in Manifest |
| ファイル選択 | `ActivityResultContracts` |

- 暗黙的Intentで他アプリとのデータ連携
- `IntentFilter`で外部からのデータ受信
- `ActivityResultContracts`で型安全なResult取得
- `createChooser`でアプリ選択UIを表示

---

8種類のAndroidアプリテンプレート（Intent連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [App Links](https://zenn.dev/myougatheaxo/articles/android-compose-app-links-2026)
- [ファイル管理](https://zenn.dev/myougatheaxo/articles/android-compose-file-manager-2026)
- [ContentProvider](https://zenn.dev/myougatheaxo/articles/android-compose-content-provider-2026)
