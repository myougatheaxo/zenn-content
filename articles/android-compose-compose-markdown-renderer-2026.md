---
title: "Compose Markdownレンダラー完全ガイド — Markdown表示/カスタムスタイル/コードハイライト"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "markdown"]
published: true
---

## この記事で学べること

**Compose Markdownレンダラー**（Markdown表示、AnnotatedString、カスタムスタイル、コードブロック）を解説します。

---

## シンプルMarkdown表示

```kotlin
// AnnotatedStringで簡易Markdown
@Composable
fun SimpleMarkdown(text: String) {
    val annotated = buildAnnotatedString {
        var remaining = text
        val boldRegex = """\*\*(.*?)\*\*""".toRegex()
        val codeRegex = """`(.*?)`""".toRegex()

        while (remaining.isNotEmpty()) {
            val boldMatch = boldRegex.find(remaining)
            val codeMatch = codeRegex.find(remaining)
            val firstMatch = listOfNotNull(boldMatch, codeMatch)
                .minByOrNull { it.range.first }

            if (firstMatch == null) {
                append(remaining); break
            }

            append(remaining.substring(0, firstMatch.range.first))

            when (firstMatch) {
                boldMatch -> withStyle(SpanStyle(fontWeight = FontWeight.Bold)) {
                    append(firstMatch.groupValues[1])
                }
                codeMatch -> withStyle(SpanStyle(
                    fontFamily = FontFamily.Monospace,
                    background = Color(0xFFE8E8E8)
                )) {
                    append(firstMatch.groupValues[1])
                }
            }
            remaining = remaining.substring(firstMatch.range.last + 1)
        }
    }

    Text(annotated, style = MaterialTheme.typography.bodyLarge)
}
```

---

## ブロック要素対応

```kotlin
@Composable
fun MarkdownRenderer(markdown: String) {
    val lines = markdown.lines()

    LazyColumn(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        items(lines) { line ->
            when {
                line.startsWith("# ") -> Text(line.drop(2),
                    style = MaterialTheme.typography.headlineLarge)
                line.startsWith("## ") -> Text(line.drop(3),
                    style = MaterialTheme.typography.headlineMedium)
                line.startsWith("### ") -> Text(line.drop(4),
                    style = MaterialTheme.typography.headlineSmall)
                line.startsWith("- ") -> Row {
                    Text("•  ")
                    Text(line.drop(2))
                }
                line.startsWith("> ") -> Row {
                    Box(Modifier.width(4.dp).height(24.dp)
                        .background(MaterialTheme.colorScheme.primary))
                    Spacer(Modifier.width(8.dp))
                    Text(line.drop(2), color = MaterialTheme.colorScheme.onSurfaceVariant,
                        fontStyle = FontStyle.Italic)
                }
                line.startsWith("---") -> HorizontalDivider()
                line.isBlank() -> Spacer(Modifier.height(4.dp))
                else -> SimpleMarkdown(line)
            }
        }
    }
}
```

---

## コードブロック表示

```kotlin
@Composable
fun CodeBlock(code: String, language: String = "") {
    ElevatedCard(
        Modifier.fillMaxWidth(),
        colors = CardDefaults.elevatedCardColors(
            containerColor = Color(0xFF1E1E1E)
        )
    ) {
        Column(Modifier.padding(16.dp)) {
            if (language.isNotEmpty()) {
                Text(language, color = Color(0xFF808080),
                    style = MaterialTheme.typography.labelSmall)
                Spacer(Modifier.height(4.dp))
            }
            SelectionContainer {
                Text(code, color = Color(0xFFD4D4D4),
                    fontFamily = FontFamily.Monospace,
                    style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| `AnnotatedString` | インラインスタイル |
| `buildAnnotatedString` | 太字/コード等 |
| ブロック解析 | 見出し/リスト/引用 |
| `SelectionContainer` | テキスト選択可能 |

- `AnnotatedString`でインラインMarkdown（太字・コード）
- 行頭の記号でブロック要素を判定
- コードブロックはモノスペースフォント+ダーク背景
- `SelectionContainer`でコードのコピーを可能に

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose RichTextEditor](https://zenn.dev/myougatheaxo/articles/android-compose-compose-rich-text-editor-2026)
- [Compose AnnotatedString](https://zenn.dev/myougatheaxo/articles/android-compose-compose-annotated-string-2026)
- [Compose WebView](https://zenn.dev/myougatheaxo/articles/android-compose-compose-webview-2026)
