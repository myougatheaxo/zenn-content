---
title: "クリップボード完全ガイド — コピー/ペースト/リッチテキスト/監視"
emoji: "📎"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "clipboard"]
published: true
---

## この記事で学べること

**クリップボード操作**（コピー/ペースト、リッチテキスト、画像コピー、クリップボード監視）を解説します。

---

## 基本コピー&ペースト

```kotlin
@Composable
fun CopyableText(text: String) {
    val clipboardManager = LocalClipboardManager.current
    val context = LocalContext.current

    Row(verticalAlignment = Alignment.CenterVertically) {
        Text(text, modifier = Modifier.weight(1f))
        IconButton(onClick = {
            clipboardManager.setText(AnnotatedString(text))
            Toast.makeText(context, "コピーしました", Toast.LENGTH_SHORT).show()
        }) {
            Icon(Icons.Default.ContentCopy, "コピー")
        }
    }
}

@Composable
fun PasteButton(onPaste: (String) -> Unit) {
    val clipboardManager = LocalClipboardManager.current

    Button(onClick = {
        clipboardManager.getText()?.text?.let { onPaste(it) }
    }) {
        Icon(Icons.Default.ContentPaste, null)
        Spacer(Modifier.width(4.dp))
        Text("ペースト")
    }
}
```

---

## システムClipboardManager

```kotlin
class ClipboardHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val clipboardManager = context.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager

    // テキストコピー
    fun copyText(label: String, text: String) {
        val clip = ClipData.newPlainText(label, text)
        clipboardManager.setPrimaryClip(clip)
    }

    // URIコピー
    fun copyUri(label: String, uri: Uri) {
        val clip = ClipData.newUri(context.contentResolver, label, uri)
        clipboardManager.setPrimaryClip(clip)
    }

    // HTMLコピー
    fun copyHtml(label: String, html: String, plainText: String) {
        val clip = ClipData.newHtmlText(label, plainText, html)
        clipboardManager.setPrimaryClip(clip)
    }

    // ペースト
    fun getText(): String? {
        val clip = clipboardManager.primaryClip ?: return null
        return clip.getItemAt(0)?.text?.toString()
    }

    // クリップボード監視
    fun observe(): Flow<String?> = callbackFlow {
        val listener = ClipboardManager.OnPrimaryClipChangedListener {
            trySend(getText())
        }
        clipboardManager.addPrimaryClipChangedListener(listener)
        awaitClose { clipboardManager.removePrimaryClipChangedListener(listener) }
    }
}
```

---

## コピー可能カード

```kotlin
@Composable
fun CopyableCard(label: String, value: String) {
    val clipboardManager = LocalClipboardManager.current
    var showCopied by remember { mutableStateOf(false) }

    Card(
        Modifier
            .fillMaxWidth()
            .clickable {
                clipboardManager.setText(AnnotatedString(value))
                showCopied = true
            }
            .padding(4.dp)
    ) {
        Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
            Column(Modifier.weight(1f)) {
                Text(label, style = MaterialTheme.typography.labelSmall)
                Text(value, style = MaterialTheme.typography.bodyLarge)
            }
            AnimatedVisibility(showCopied) {
                Icon(Icons.Default.Check, "コピー済み", tint = MaterialTheme.colorScheme.primary)
            }
        }
    }

    LaunchedEffect(showCopied) {
        if (showCopied) { delay(2000); showCopied = false }
    }
}
```

---

## まとめ

| 操作 | API |
|------|-----|
| テキストコピー | `ClipData.newPlainText()` |
| HTMLコピー | `ClipData.newHtmlText()` |
| URIコピー | `ClipData.newUri()` |
| Compose | `LocalClipboardManager` |

- `LocalClipboardManager`でCompose内のコピー/ペースト
- `ClipboardManager`で高度なクリップボード操作
- `callbackFlow`でクリップボード変更をリアクティブ監視
- コピー完了のフィードバックUIを忘れずに

---

8種類のAndroidアプリテンプレート（クリップボード対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Share Intent](https://zenn.dev/myougatheaxo/articles/android-compose-share-intent-2026)
- [テキスト計測](https://zenn.dev/myougatheaxo/articles/android-compose-text-measurement-2026)
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-gesture-touch-2026)
