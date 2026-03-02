---
title: "クリップボード/コピー機能ガイド — Compose版"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "clipboard"]
published: true
---

## この記事で学べること

Composeでの**クリップボード操作**（コピー、ペースト、SelectionContainer、カスタムコピーボタン）を解説します。

---

## テキストコピー

```kotlin
@Composable
fun CopyButton(text: String) {
    val clipboardManager = LocalClipboardManager.current
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    IconButton(onClick = {
        clipboardManager.setText(AnnotatedString(text))
        scope.launch {
            snackbarHostState.showSnackbar("コピーしました")
        }
    }) {
        Icon(Icons.Default.ContentCopy, contentDescription = "コピー")
    }
}
```

---

## コピー可能なテキスト

```kotlin
@Composable
fun CopyableText(text: String, modifier: Modifier = Modifier) {
    val clipboardManager = LocalClipboardManager.current

    Row(
        modifier = modifier.fillMaxWidth(),
        verticalAlignment = Alignment.CenterVertically
    ) {
        SelectionContainer(Modifier.weight(1f)) {
            Text(text, style = MaterialTheme.typography.bodyLarge)
        }

        IconButton(onClick = {
            clipboardManager.setText(AnnotatedString(text))
        }) {
            Icon(
                Icons.Default.ContentCopy,
                contentDescription = "コピー",
                modifier = Modifier.size(20.dp)
            )
        }
    }
}

// コードブロック風
@Composable
fun CodeBlock(code: String) {
    val clipboardManager = LocalClipboardManager.current

    Card(
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.surfaceVariant
        )
    ) {
        Box(Modifier.fillMaxWidth()) {
            SelectionContainer {
                Text(
                    text = code,
                    fontFamily = FontFamily.Monospace,
                    fontSize = 14.sp,
                    modifier = Modifier.padding(16.dp).padding(end = 32.dp)
                )
            }

            IconButton(
                onClick = { clipboardManager.setText(AnnotatedString(code)) },
                modifier = Modifier.align(Alignment.TopEnd).size(32.dp)
            ) {
                Icon(Icons.Default.ContentCopy, null, Modifier.size(16.dp))
            }
        }
    }
}
```

---

## クリップボードからペースト

```kotlin
@Composable
fun PasteField() {
    val clipboardManager = LocalClipboardManager.current
    var text by remember { mutableStateOf("") }

    OutlinedTextField(
        value = text,
        onValueChange = { text = it },
        label = { Text("テキスト入力") },
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
```

---

## SelectionContainer（範囲選択）

```kotlin
@Composable
fun SelectableArticle() {
    // この中のテキストはすべて選択可能
    SelectionContainer {
        Column(Modifier.padding(16.dp)) {
            Text(
                "記事タイトル",
                style = MaterialTheme.typography.headlineMedium
            )
            Spacer(Modifier.height(8.dp))
            Text(
                "記事の本文がここに入ります。ユーザーはこのテキストを長押しして選択し、コピーできます。",
                style = MaterialTheme.typography.bodyLarge
            )

            // 選択不可にしたい部分
            DisableSelection {
                Text(
                    "この部分は選択できません",
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }
    }
}
```

---

## APIキー/URLのコピーUI

```kotlin
@Composable
fun ApiKeyDisplay(apiKey: String) {
    val clipboardManager = LocalClipboardManager.current
    var copied by remember { mutableStateOf(false) }

    OutlinedCard(Modifier.fillMaxWidth()) {
        Row(
            Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text(
                text = apiKey.take(8) + "..." + apiKey.takeLast(4),
                fontFamily = FontFamily.Monospace,
                modifier = Modifier.weight(1f)
            )

            AnimatedContent(targetState = copied, label = "copy") { isCopied ->
                IconButton(onClick = {
                    clipboardManager.setText(AnnotatedString(apiKey))
                    copied = true
                }) {
                    Icon(
                        if (isCopied) Icons.Default.Check else Icons.Default.ContentCopy,
                        null,
                        tint = if (isCopied) Color(0xFF4CAF50)
                            else MaterialTheme.colorScheme.onSurface
                    )
                }
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

- `LocalClipboardManager.current`でクリップボード操作
- `setText(AnnotatedString(text))`でコピー
- `getText()?.text`でペースト
- `SelectionContainer`で範囲選択可能化
- `DisableSelection`で部分的に選択不可
- コピー後のフィードバック表示でUX向上

---

8種類のAndroidアプリテンプレート（クリップボード機能実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [共有/インテント](https://zenn.dev/myougatheaxo/articles/android-compose-share-intent-2026)
- [TextField詳細](https://zenn.dev/myougatheaxo/articles/android-compose-text-field-advanced-2026)
- [Snackbar/Toast](https://zenn.dev/myougatheaxo/articles/android-compose-snackbar-toast-2026)
