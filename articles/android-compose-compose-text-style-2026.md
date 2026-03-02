---
title: "TextStyle完全ガイド — AnnotatedString/SpanStyle/テキスト装飾/リンク"
emoji: "✒️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**TextStyle**（AnnotatedString、SpanStyle、テキスト装飾、クリッカブルリンク）を解説します。

---

## AnnotatedString

```kotlin
@Composable
fun StyledText() {
    val annotated = buildAnnotatedString {
        append("これは")
        withStyle(SpanStyle(color = Color.Red, fontWeight = FontWeight.Bold)) {
            append("重要")
        }
        append("なテキストです。")
        withStyle(SpanStyle(
            textDecoration = TextDecoration.Underline,
            color = MaterialTheme.colorScheme.primary
        )) {
            append("リンク")
        }
        append("もあります。")
    }

    Text(annotated, fontSize = 18.sp)
}
```

---

## クリッカブルテキスト

```kotlin
@Composable
fun ClickableLink() {
    val uriHandler = LocalUriHandler.current
    val annotated = buildAnnotatedString {
        append("利用規約に")
        pushStringAnnotation(tag = "URL", annotation = "https://example.com/terms")
        withStyle(SpanStyle(
            color = MaterialTheme.colorScheme.primary,
            textDecoration = TextDecoration.Underline
        )) {
            append("同意")
        }
        pop()
        append("します。")
    }

    ClickableText(
        text = annotated,
        onClick = { offset ->
            annotated.getStringAnnotations("URL", offset, offset)
                .firstOrNull()?.let { uriHandler.openUri(it.item) }
        },
        style = MaterialTheme.typography.bodyLarge
    )
}
```

---

## テキスト装飾

```kotlin
@Composable
fun TextDecorations() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("太字", fontWeight = FontWeight.Bold)
        Text("イタリック", fontStyle = FontStyle.Italic)
        Text("下線", textDecoration = TextDecoration.Underline)
        Text("取り消し線", textDecoration = TextDecoration.LineThrough)
        Text("影付き", style = TextStyle(shadow = Shadow(Color.Gray, Offset(2f, 2f), 4f)))
        Text("文字間隔", letterSpacing = 4.sp)
        Text(
            "グラデーション",
            style = TextStyle(
                brush = Brush.linearGradient(listOf(Color.Red, Color.Blue)),
                fontSize = 24.sp
            )
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AnnotatedString` | 部分スタイル |
| `SpanStyle` | インラインスタイル |
| `ParagraphStyle` | 段落スタイル |
| `ClickableText` | リンクテキスト |

- `buildAnnotatedString`で部分的にスタイル適用
- `SpanStyle`で色/太字/下線を部分指定
- `ClickableText`でURLリンク実装
- `Brush`でグラデーションテキスト

---

8種類のAndroidアプリテンプレート（UI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Typography](https://zenn.dev/myougatheaxo/articles/android-compose-compose-typography-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [テキスト選択](https://zenn.dev/myougatheaxo/articles/android-compose-text-selection-2026)
