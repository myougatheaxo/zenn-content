---
title: "Compose RichText完全ガイド — AnnotatedString/SpanStyle/InlineContent/リンク付きテキスト"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "text"]
published: true
---

## この記事で学べること

**Compose RichText**（AnnotatedString、SpanStyle、InlineContent、クリック可能テキスト）を解説します。

---

## AnnotatedString

```kotlin
@Composable
fun RichTextDemo() {
    val annotatedText = buildAnnotatedString {
        withStyle(SpanStyle(fontSize = 24.sp, fontWeight = FontWeight.Bold)) {
            append("Compose ")
        }
        withStyle(SpanStyle(color = Color(0xFF6200EE), fontSize = 24.sp)) {
            append("RichText")
        }
        append("\n\n")
        append("通常テキストに")
        withStyle(SpanStyle(fontWeight = FontWeight.Bold)) { append("太字") }
        append("、")
        withStyle(SpanStyle(fontStyle = FontStyle.Italic)) { append("イタリック") }
        append("、")
        withStyle(SpanStyle(textDecoration = TextDecoration.Underline)) { append("下線") }
        append("、")
        withStyle(SpanStyle(color = Color.Red)) { append("赤色") }
        append("を混在できます。")
    }

    Text(annotatedText, modifier = Modifier.padding(16.dp))
}
```

---

## クリック可能リンク

```kotlin
@Composable
fun LinkedText() {
    val uriHandler = LocalUriHandler.current
    val annotatedText = buildAnnotatedString {
        append("利用規約については")
        pushStringAnnotation(tag = "URL", annotation = "https://example.com/terms")
        withStyle(SpanStyle(color = Color.Blue, textDecoration = TextDecoration.Underline)) {
            append("こちら")
        }
        pop()
        append("をご確認ください。")
    }

    ClickableText(
        text = annotatedText,
        style = MaterialTheme.typography.bodyLarge,
        onClick = { offset ->
            annotatedText.getStringAnnotations("URL", offset, offset)
                .firstOrNull()?.let { uriHandler.openUri(it.item) }
        }
    )
}
```

---

## InlineContent

```kotlin
@Composable
fun TextWithInlineIcon() {
    val id = "icon"
    val text = buildAnnotatedString {
        append("評価: ")
        repeat(5) {
            appendInlineContent(id)
        }
        append(" (4.5)")
    }

    val inlineContent = mapOf(
        id to InlineTextContent(
            Placeholder(20.sp, 20.sp, PlaceholderVerticalAlign.TextCenter)
        ) {
            Icon(Icons.Default.Star, contentDescription = null,
                tint = Color(0xFFFFB300), modifier = Modifier.fillMaxSize())
        }
    )

    Text(text, inlineContent = inlineContent, style = MaterialTheme.typography.titleMedium,
        modifier = Modifier.padding(16.dp))
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `buildAnnotatedString` | リッチテキスト構築 |
| `SpanStyle` | テキストスタイル |
| `ClickableText` | クリック検出 |
| `InlineTextContent` | インラインコンテンツ |

- `buildAnnotatedString`でリッチテキストを構築
- `SpanStyle`で色、太字、下線等を部分適用
- `ClickableText`でリンククリックを検出
- `InlineTextContent`でテキスト内にアイコン配置

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TextSelection](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-selection-2026)
- [Compose TextStyle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
- [Compose Typography](https://zenn.dev/myougatheaxo/articles/android-compose-compose-typography-2026)
