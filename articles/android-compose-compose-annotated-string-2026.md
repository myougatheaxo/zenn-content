---
title: "Compose AnnotatedString完全ガイド — リッチテキスト/クリック可能テキスト/スタイル"
emoji: "✍️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "text"]
published: true
---

## この記事で学べること

**Compose AnnotatedString**（リッチテキスト、部分スタイル、クリック可能テキスト、URLリンク）を解説します。

---

## 基本AnnotatedString

```kotlin
@Composable
fun RichTextDemo() {
    Text(buildAnnotatedString {
        append("これは")
        withStyle(SpanStyle(fontWeight = FontWeight.Bold)) { append("太字") }
        append("と")
        withStyle(SpanStyle(color = Color.Red, fontSize = 20.sp)) { append("赤い大きな") }
        append("テキストです。")
    })
}

@Composable
fun MultiStyleText() {
    Text(buildAnnotatedString {
        withStyle(SpanStyle(color = Color(0xFF2196F3), fontWeight = FontWeight.Bold)) {
            append("Info: ")
        }
        append("アプリが更新されました。")
        withStyle(SpanStyle(textDecoration = TextDecoration.Underline, color = Color(0xFF4CAF50))) {
            append("詳細を見る")
        }
    })
}
```

---

## クリック可能テキスト

```kotlin
@Composable
fun ClickableRichText() {
    val annotatedText = buildAnnotatedString {
        append("利用規約に同意する場合は")
        pushStringAnnotation(tag = "URL", annotation = "https://example.com/terms")
        withStyle(SpanStyle(color = Color.Blue, textDecoration = TextDecoration.Underline)) {
            append("利用規約")
        }
        pop()
        append("と")
        pushStringAnnotation(tag = "URL", annotation = "https://example.com/privacy")
        withStyle(SpanStyle(color = Color.Blue, textDecoration = TextDecoration.Underline)) {
            append("プライバシーポリシー")
        }
        pop()
        append("をご確認ください。")
    }

    val uriHandler = LocalUriHandler.current

    ClickableText(
        text = annotatedText,
        onClick = { offset ->
            annotatedText.getStringAnnotations("URL", offset, offset).firstOrNull()?.let {
                uriHandler.openUri(it.item)
            }
        },
        style = MaterialTheme.typography.bodyMedium
    )
}
```

---

## パラグラフスタイル

```kotlin
@Composable
fun ParagraphStyleDemo() {
    Text(buildAnnotatedString {
        withStyle(ParagraphStyle(textAlign = TextAlign.Center)) {
            withStyle(SpanStyle(fontSize = 24.sp, fontWeight = FontWeight.Bold)) {
                append("タイトル\n")
            }
        }
        withStyle(ParagraphStyle(textIndent = TextIndent(firstLine = 20.sp))) {
            append("本文テキストがここに入ります。最初の行はインデントされます。")
        }
    })
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `buildAnnotatedString` | リッチテキスト構築 |
| `SpanStyle` | 文字スタイル |
| `ParagraphStyle` | 段落スタイル |
| `pushStringAnnotation` | クリック領域 |

- `buildAnnotatedString`で部分的にスタイル変更
- `pushStringAnnotation`+`ClickableText`でリンク実装
- `ParagraphStyle`で段落単位のスタイル適用
- `TextIndent`で最初の行のインデント設定

---

8種類のAndroidアプリテンプレート（リッチテキスト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TextDecoration](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-decoration-2026)
- [Compose TextStyle](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
- [Compose RichTextEditor](https://zenn.dev/myougatheaxo/articles/android-compose-compose-rich-text-editor-2026)
