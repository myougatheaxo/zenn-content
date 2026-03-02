---
title: "Compose HTMLText完全ガイド — HTML→AnnotatedString/リンク処理/スタイル変換"
emoji: "🌐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "html"]
published: true
---

## この記事で学べること

**Compose HTMLText**（HTMLパース、AnnotatedString変換、リンク処理、カスタムスタイル）を解説します。

---

## HTML→AnnotatedString変換

```kotlin
@Composable
fun HtmlText(html: String, modifier: Modifier = Modifier) {
    val uriHandler = LocalUriHandler.current
    val annotatedString = remember(html) {
        buildAnnotatedString {
            val spanned = HtmlCompat.fromHtml(html, HtmlCompat.FROM_HTML_MODE_COMPACT)
            append(spanned.toString())

            // スパンの適用
            spanned.getSpans(0, spanned.length, Any::class.java).forEach { span ->
                val start = spanned.getSpanStart(span)
                val end = spanned.getSpanEnd(span)
                when (span) {
                    is StyleSpan -> when (span.style) {
                        Typeface.BOLD -> addStyle(SpanStyle(fontWeight = FontWeight.Bold), start, end)
                        Typeface.ITALIC -> addStyle(SpanStyle(fontStyle = FontStyle.Italic), start, end)
                    }
                    is UnderlineSpan ->
                        addStyle(SpanStyle(textDecoration = TextDecoration.Underline), start, end)
                    is URLSpan -> {
                        addStyle(SpanStyle(color = Color.Blue, textDecoration = TextDecoration.Underline), start, end)
                        addStringAnnotation("URL", span.url, start, end)
                    }
                }
            }
        }
    }

    ClickableText(
        text = annotatedString,
        modifier = modifier,
        style = MaterialTheme.typography.bodyLarge,
        onClick = { offset ->
            annotatedString.getStringAnnotations("URL", offset, offset)
                .firstOrNull()?.let { uriHandler.openUri(it.item) }
        }
    )
}
```

---

## 使用例

```kotlin
@Composable
fun HtmlContentScreen() {
    val htmlContent = """
        <h2>Jetpack Compose</h2>
        <p><b>宣言的UI</b>ツールキットです。</p>
        <p>詳細は<a href="https://developer.android.com">公式サイト</a>を参照。</p>
        <ul>
            <li>Kotlin製</li>
            <li><i>リアクティブ</i></li>
        </ul>
    """.trimIndent()

    Column(Modifier.fillMaxSize().padding(16.dp).verticalScroll(rememberScrollState())) {
        HtmlText(html = htmlContent)
    }
}
```

---

## APIレスポンス表示

```kotlin
@Composable
fun ArticleContent(article: Article) {
    Column(Modifier.padding(16.dp)) {
        Text(article.title, style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(8.dp))
        Text("${article.date} · ${article.author}",
            color = MaterialTheme.colorScheme.onSurfaceVariant)
        HorizontalDivider(Modifier.padding(vertical = 16.dp))

        // サーバーから返されたHTML本文を表示
        HtmlText(html = article.htmlBody, modifier = Modifier.fillMaxWidth())
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `HtmlCompat.fromHtml` | HTMLパース |
| `AnnotatedString` | リッチテキスト |
| `ClickableText` | リンククリック |
| `SpanStyle` | スタイル適用 |

- `HtmlCompat.fromHtml`でHTML→Spanned変換
- Spannedから`AnnotatedString`に変換してCompose表示
- URLSpanを検出してクリック可能なリンクに
- APIレスポンスのHTMLコンテンツ表示に最適

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose RichText](https://zenn.dev/myougatheaxo/articles/android-compose-compose-richtext-2026)
- [Compose Markdown](https://zenn.dev/myougatheaxo/articles/android-compose-compose-markdown-2026)
- [Compose WebView](https://zenn.dev/myougatheaxo/articles/android-compose-compose-webview-2026)
