---
title: "リッチテキストエディタ完全ガイド — BasicTextField/AnnotatedString/Markdown"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "texteditor"]
published: true
---

## この記事で学べること

**リッチテキストエディタ**（BasicTextField、AnnotatedString、太字/斜体/リンク、Markdownプレビュー、ツールバー）を解説します。

---

## AnnotatedStringベースのエディタ

```kotlin
@Composable
fun RichTextEditor() {
    var text by remember { mutableStateOf(TextFieldValue("")) }
    var isBold by remember { mutableStateOf(false) }
    var isItalic by remember { mutableStateOf(false) }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        // ツールバー
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            IconToggleButton(checked = isBold, onCheckedChange = { isBold = it }) {
                Icon(Icons.Default.FormatBold, "太字",
                    tint = if (isBold) MaterialTheme.colorScheme.primary
                           else MaterialTheme.colorScheme.onSurface)
            }
            IconToggleButton(checked = isItalic, onCheckedChange = { isItalic = it }) {
                Icon(Icons.Default.FormatItalic, "斜体",
                    tint = if (isItalic) MaterialTheme.colorScheme.primary
                           else MaterialTheme.colorScheme.onSurface)
            }
        }

        HorizontalDivider()
        Spacer(Modifier.height(8.dp))

        BasicTextField(
            value = text,
            onValueChange = { text = it },
            modifier = Modifier.fillMaxSize(),
            textStyle = TextStyle(
                fontWeight = if (isBold) FontWeight.Bold else FontWeight.Normal,
                fontStyle = if (isItalic) FontStyle.Italic else FontStyle.Normal,
                fontSize = 16.sp
            )
        )
    }
}
```

---

## Markdownプレビュー

```kotlin
@Composable
fun MarkdownPreview(markdown: String) {
    val annotatedString = remember(markdown) {
        parseMarkdown(markdown)
    }
    Text(annotatedString, Modifier.padding(16.dp))
}

fun parseMarkdown(text: String): AnnotatedString {
    return buildAnnotatedString {
        text.lines().forEach { line ->
            when {
                line.startsWith("# ") -> {
                    withStyle(SpanStyle(fontSize = 24.sp, fontWeight = FontWeight.Bold)) {
                        append(line.removePrefix("# "))
                    }
                }
                line.startsWith("## ") -> {
                    withStyle(SpanStyle(fontSize = 20.sp, fontWeight = FontWeight.Bold)) {
                        append(line.removePrefix("## "))
                    }
                }
                line.startsWith("- ") -> {
                    append("  • ${line.removePrefix("- ")}")
                }
                else -> {
                    // **太字** と *斜体*
                    var remaining = line
                    val boldRegex = Regex("\\*\\*(.*?)\\*\\*")
                    val italicRegex = Regex("\\*(.*?)\\*")

                    boldRegex.findAll(remaining).forEach { match ->
                        val before = remaining.substring(0, match.range.first)
                        append(before)
                        withStyle(SpanStyle(fontWeight = FontWeight.Bold)) {
                            append(match.groupValues[1])
                        }
                        remaining = remaining.substring(match.range.last + 1)
                    }
                    append(remaining)
                }
            }
            append("\n")
        }
    }
}
```

---

## エディタ+プレビュー切替

```kotlin
@Composable
fun MarkdownEditor() {
    var text by rememberSaveable { mutableStateOf("") }
    var isPreview by remember { mutableStateOf(false) }

    Column(Modifier.fillMaxSize()) {
        Row(Modifier.fillMaxWidth().padding(8.dp)) {
            FilterChip(selected = !isPreview, onClick = { isPreview = false },
                label = { Text("編集") })
            Spacer(Modifier.width(8.dp))
            FilterChip(selected = isPreview, onClick = { isPreview = true },
                label = { Text("プレビュー") })
        }

        if (isPreview) {
            MarkdownPreview(text)
        } else {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                modifier = Modifier.fillMaxSize().padding(8.dp),
                placeholder = { Text("Markdownで入力...") }
            )
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 基本エディタ | `BasicTextField` |
| リッチテキスト | `AnnotatedString` |
| Markdown解析 | カスタムパーサー |
| ツールバー | `IconToggleButton` |

- `BasicTextField`でカスタムテキスト入力
- `AnnotatedString`で太字/斜体/色付き表示
- Markdown→AnnotatedString変換でプレビュー
- 編集/プレビュー切替でエディタUX向上

---

8種類のAndroidアプリテンプレート（テキスト編集対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキスト入力](https://zenn.dev/myougatheaxo/articles/android-compose-text-field-advanced-2026)
- [テキスト測定](https://zenn.dev/myougatheaxo/articles/android-compose-text-measurement-2026)
- [クリップボード](https://zenn.dev/myougatheaxo/articles/android-compose-clipboard-copy-2026)
