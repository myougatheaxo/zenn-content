---
title: "Clipboard完全ガイド — ClipboardManager/コピー/ペースト/リッチテキスト"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Clipboard**（ClipboardManager、テキストコピー、ペースト、クリップボード監視）を解説します。

---

## テキストコピー

```kotlin
@Composable
fun CopyToClipboard() {
    val context = LocalContext.current
    val clipboardManager = LocalClipboardManager.current
    val textToCopy = "コピーするテキスト"

    Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
        Text(textToCopy, Modifier.weight(1f))
        IconButton(onClick = {
            clipboardManager.setText(AnnotatedString(textToCopy))
            Toast.makeText(context, "コピーしました", Toast.LENGTH_SHORT).show()
        }) {
            Icon(Icons.Default.ContentCopy, "コピー")
        }
    }
}
```

---

## ペースト

```kotlin
@Composable
fun PasteFromClipboard() {
    val clipboardManager = LocalClipboardManager.current
    var text by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("入力") },
            trailingIcon = {
                IconButton(onClick = {
                    clipboardManager.getText()?.let { text = it.text }
                }) {
                    Icon(Icons.Default.ContentPaste, "ペースト")
                }
            },
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## コピー可能カード

```kotlin
@Composable
fun CopyableInfoCard(label: String, value: String) {
    val clipboardManager = LocalClipboardManager.current
    val context = LocalContext.current
    var copied by remember { mutableStateOf(false) }

    Card(Modifier.fillMaxWidth()) {
        Row(
            Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column(Modifier.weight(1f)) {
                Text(label, fontSize = 12.sp, color = MaterialTheme.colorScheme.outline)
                Text(value, style = MaterialTheme.typography.bodyLarge)
            }
            IconButton(onClick = {
                clipboardManager.setText(AnnotatedString(value))
                copied = true
                Toast.makeText(context, "コピーしました", Toast.LENGTH_SHORT).show()
            }) {
                Icon(
                    if (copied) Icons.Default.Check else Icons.Default.ContentCopy,
                    "コピー"
                )
            }
        }
    }

    LaunchedEffect(copied) {
        if (copied) { delay(2000); copied = false }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LocalClipboardManager` | Compose用クリップボード |
| `setText` | テキストコピー |
| `getText` | テキスト取得 |
| `ClipboardManager` | Android標準API |

- `LocalClipboardManager`でCompose内からクリップボード操作
- `setText(AnnotatedString())`でテキストコピー
- `getText()`でペースト
- コピー後にアイコン変更でフィードバック

---

8種類のAndroidアプリテンプレート（ユーティリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキスト選択](https://zenn.dev/myougatheaxo/articles/android-compose-text-selection-2026)
- [ShareSheet](https://zenn.dev/myougatheaxo/articles/android-compose-compose-share-sheet-2026)
- [Snackbar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-snackbar-2026)
