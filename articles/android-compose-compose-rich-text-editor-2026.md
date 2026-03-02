---
title: "RichTextEditor完全ガイド — AnnotatedString/スタイル切替/マークダウンプレビュー"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "textfield"]
published: true
---

## この記事で学べること

**RichTextEditor**（AnnotatedString編集、スタイル切替ツールバー、マークダウンプレビュー）を解説します。

---

## スタイル切替

```kotlin
@Composable
fun RichTextEditor() {
    var text by remember { mutableStateOf("") }
    var isBold by remember { mutableStateOf(false) }
    var isItalic by remember { mutableStateOf(false) }
    var isUnderline by remember { mutableStateOf(false) }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        // ツールバー
        Row(horizontalArrangement = Arrangement.spacedBy(4.dp)) {
            IconToggleButton(checked = isBold, onCheckedChange = { isBold = it }) {
                Icon(Icons.Default.FormatBold, "太字",
                    tint = if (isBold) MaterialTheme.colorScheme.primary else Color.Gray)
            }
            IconToggleButton(checked = isItalic, onCheckedChange = { isItalic = it }) {
                Icon(Icons.Default.FormatItalic, "斜体",
                    tint = if (isItalic) MaterialTheme.colorScheme.primary else Color.Gray)
            }
            IconToggleButton(checked = isUnderline, onCheckedChange = { isUnderline = it }) {
                Icon(Icons.Default.FormatUnderlined, "下線",
                    tint = if (isUnderline) MaterialTheme.colorScheme.primary else Color.Gray)
            }
        }

        HorizontalDivider(Modifier.padding(vertical = 8.dp))

        // エディタ
        OutlinedTextField(
            value = text,
            onValueChange = { text = it },
            textStyle = TextStyle(
                fontWeight = if (isBold) FontWeight.Bold else FontWeight.Normal,
                fontStyle = if (isItalic) FontStyle.Italic else FontStyle.Normal,
                textDecoration = if (isUnderline) TextDecoration.Underline else TextDecoration.None,
                fontSize = 16.sp
            ),
            modifier = Modifier.fillMaxWidth().weight(1f),
            placeholder = { Text("テキストを入力...") }
        )
    }
}
```

---

## マークダウンプレビュー

```kotlin
@Composable
fun MarkdownEditor() {
    var markdown by remember { mutableStateOf("# タイトル\n\n**太字**と*斜体*\n\n- リスト1\n- リスト2") }
    var isPreview by remember { mutableStateOf(false) }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.End) {
            TextButton(onClick = { isPreview = !isPreview }) {
                Text(if (isPreview) "編集" else "プレビュー")
            }
        }

        if (isPreview) {
            MarkdownPreview(markdown, Modifier.weight(1f))
        } else {
            OutlinedTextField(
                value = markdown, onValueChange = { markdown = it },
                modifier = Modifier.fillMaxWidth().weight(1f),
                textStyle = TextStyle(fontFamily = FontFamily.Monospace)
            )
        }
    }
}

@Composable
fun MarkdownPreview(markdown: String, modifier: Modifier) {
    val annotated = remember(markdown) { parseMarkdown(markdown) }
    Text(annotated, modifier.verticalScroll(rememberScrollState()))
}

fun parseMarkdown(md: String): AnnotatedString = buildAnnotatedString {
    md.lines().forEach { line ->
        when {
            line.startsWith("# ") -> {
                withStyle(SpanStyle(fontSize = 24.sp, fontWeight = FontWeight.Bold)) {
                    append(line.removePrefix("# "))
                }
            }
            line.startsWith("- ") -> {
                append("  • ${line.removePrefix("- ")}")
            }
            else -> {
                var remaining = line
                while (remaining.isNotEmpty()) {
                    val boldMatch = Regex("\\*\\*(.+?)\\*\\*").find(remaining)
                    val italicMatch = Regex("\\*(.+?)\\*").find(remaining)
                    if (boldMatch != null && boldMatch.range.first == 0) {
                        withStyle(SpanStyle(fontWeight = FontWeight.Bold)) { append(boldMatch.groupValues[1]) }
                        remaining = remaining.substring(boldMatch.range.last + 1)
                    } else {
                        append(remaining.first())
                        remaining = remaining.drop(1)
                    }
                }
            }
        }
        append("\n")
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| スタイル切替 | `IconToggleButton` + `TextStyle` |
| AnnotatedString | 部分スタイル適用 |
| MDプレビュー | パース+表示切替 |
| ツールバー | Bold/Italic/Underline |

- `TextStyle`でリアルタイムスタイル切替
- `AnnotatedString`で部分的なスタイル適用
- マークダウン→AnnotatedString変換でプレビュー
- ツールバーで直感的な操作

---

8種類のAndroidアプリテンプレート（エディタ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキストスタイル](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
- [TextDecoration](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-decoration-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
