---
title: "クリップボード/テキスト選択ガイド — コピー/貼り付け/共有"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "clipboard"]
published: true
---

## この記事で学べること

Composeでの**クリップボード操作**（コピー、貼り付け、テキスト選択、共有）を解説します。

---

## クリップボードへのコピー

```kotlin
@Composable
fun CopyToClipboard() {
    val clipboardManager = LocalClipboardManager.current
    val text = "コピーするテキスト"
    var showCopied by remember { mutableStateOf(false) }

    Row(verticalAlignment = Alignment.CenterVertically) {
        Text(text, modifier = Modifier.weight(1f))
        IconButton(onClick = {
            clipboardManager.setText(AnnotatedString(text))
            showCopied = true
        }) {
            Icon(Icons.Default.ContentCopy, "コピー")
        }
    }

    if (showCopied) {
        LaunchedEffect(Unit) {
            delay(2000)
            showCopied = false
        }
        Text(
            "コピーしました",
            color = MaterialTheme.colorScheme.primary,
            style = MaterialTheme.typography.bodySmall
        )
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

    Column {
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            label = { Text("入力") },
            trailingIcon = {
                IconButton(onClick = {
                    clipboardManager.getText()?.let {
                        text = it.text
                    }
                }) {
                    Icon(Icons.Default.ContentPaste, "貼り付け")
                }
            },
            modifier = Modifier.fillMaxWidth()
        )
    }
}
```

---

## テキスト選択

```kotlin
@Composable
fun SelectableTextExample() {
    // 選択可能なテキスト
    SelectionContainer {
        Column {
            Text("このテキストは選択できます")
            Text("このテキストも選択できます")

            // 選択不可のテキストを混ぜる
            DisableSelection {
                Text("このテキストは選択できません")
            }

            Text("ここはまた選択できます")
        }
    }
}
```

---

## 共有インテント

```kotlin
@Composable
fun ShareButton(text: String) {
    val context = LocalContext.current

    Button(onClick = {
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_TEXT, text)
        }
        context.startActivity(Intent.createChooser(intent, "共有"))
    }) {
        Icon(Icons.Default.Share, null)
        Spacer(Modifier.width(8.dp))
        Text("共有")
    }
}

// URL共有
@Composable
fun ShareUrl(title: String, url: String) {
    val context = LocalContext.current

    IconButton(onClick = {
        val intent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_SUBJECT, title)
            putExtra(Intent.EXTRA_TEXT, "$title\n$url")
        }
        context.startActivity(Intent.createChooser(intent, null))
    }) {
        Icon(Icons.Default.Share, "共有")
    }
}
```

---

## コピー可能なコードブロック

```kotlin
@Composable
fun CopyableCodeBlock(code: String) {
    val clipboardManager = LocalClipboardManager.current
    var copied by remember { mutableStateOf(false) }

    Surface(
        color = MaterialTheme.colorScheme.surfaceVariant,
        shape = RoundedCornerShape(8.dp),
        modifier = Modifier.fillMaxWidth()
    ) {
        Column(Modifier.padding(12.dp)) {
            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text("Code", style = MaterialTheme.typography.labelSmall)
                TextButton(onClick = {
                    clipboardManager.setText(AnnotatedString(code))
                    copied = true
                }) {
                    Text(if (copied) "Copied!" else "Copy")
                }
            }

            SelectionContainer {
                Text(
                    code,
                    fontFamily = FontFamily.Monospace,
                    style = MaterialTheme.typography.bodySmall
                )
            }
        }
    }

    LaunchedEffect(copied) {
        if (copied) {
            delay(2000)
            copied = false
        }
    }
}
```

---

## まとめ

- `LocalClipboardManager`でコピー/貼り付け
- `setText(AnnotatedString(...))`でクリップボードに書き込み
- `getText()`でクリップボードから読み取り
- `SelectionContainer`/`DisableSelection`でテキスト選択制御
- `Intent.ACTION_SEND`で外部アプリへの共有
- コードブロックのコピーボタン実装

---

8種類のAndroidアプリテンプレート（クリップボード対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキスト装飾ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-text-styling-2026)
- [コンテキストメニューガイド](https://zenn.dev/myougatheaxo/articles/android-compose-context-menu-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
