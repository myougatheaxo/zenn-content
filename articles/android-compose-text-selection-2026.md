---
title: "テキスト選択/コピー完全ガイド — SelectionContainer/リッチテキスト/AnnotatedString"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**テキスト選択**（SelectionContainer、AnnotatedString、リッチテキスト表示、リンク付きテキスト）を解説します。

---

## SelectionContainer

```kotlin
@Composable
fun SelectableTextExample() {
    SelectionContainer {
        Column(Modifier.padding(16.dp)) {
            Text("このテキストは選択可能です", style = MaterialTheme.typography.bodyLarge)
            Spacer(Modifier.height(8.dp))
            Text("複数行にわたって選択できます", style = MaterialTheme.typography.bodyMedium)

            DisableSelection {
                Text("このテキストは選択不可", color = Color.Gray)
            }

            Text("ここはまた選択可能", style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

---

## AnnotatedString（リッチテキスト）

```kotlin
@Composable
fun RichTextExample() {
    val annotatedString = buildAnnotatedString {
        append("利用規約に")

        pushStringAnnotation(tag = "URL", annotation = "https://example.com/terms")
        withStyle(SpanStyle(color = MaterialTheme.colorScheme.primary, textDecoration = TextDecoration.Underline)) {
            append("同意")
        }
        pop()

        append("して続行します。")
    }

    ClickableText(
        text = annotatedString,
        style = MaterialTheme.typography.bodyLarge,
        onClick = { offset ->
            annotatedString.getStringAnnotations("URL", offset, offset).firstOrNull()?.let { annotation ->
                // URLを開く
            }
        }
    )
}
```

---

## コピー機能付きテキスト

```kotlin
@Composable
fun CopyableText(text: String, label: String) {
    val clipboardManager = LocalClipboardManager.current
    var copied by remember { mutableStateOf(false) }

    Row(
        Modifier
            .fillMaxWidth()
            .clip(RoundedCornerShape(8.dp))
            .background(MaterialTheme.colorScheme.surfaceVariant)
            .clickable {
                clipboardManager.setText(AnnotatedString(text))
                copied = true
            }
            .padding(12.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Column(Modifier.weight(1f)) {
            Text(label, style = MaterialTheme.typography.labelSmall, color = Color.Gray)
            Text(text, style = MaterialTheme.typography.bodyMedium)
        }
        Icon(
            if (copied) Icons.Default.Check else Icons.Default.ContentCopy,
            contentDescription = "コピー",
            tint = if (copied) Color(0xFF4CAF50) else LocalContentColor.current
        )
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

| 機能 | API |
|------|-----|
| 選択可能テキスト | `SelectionContainer` |
| 選択無効化 | `DisableSelection` |
| リッチテキスト | `AnnotatedString` |
| クリップボード | `LocalClipboardManager` |

- `SelectionContainer`でテキスト選択を有効化
- `AnnotatedString`でリンク付きリッチテキスト
- `LocalClipboardManager`でコピー機能実装
- `DisableSelection`で部分的に選択無効化

---

8種類のAndroidアプリテンプレート（テキスト機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [クリップボード](https://zenn.dev/myougatheaxo/articles/android-compose-clipboard-2026)
- [キーボード](https://zenn.dev/myougatheaxo/articles/android-compose-keyboard-shortcut-2026)
- [TextField](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
