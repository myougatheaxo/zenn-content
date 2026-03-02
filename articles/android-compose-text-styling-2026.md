---
title: "テキスト装飾ガイド — ComposeでAnnotatedString/SpanStyle/リッチテキスト"
emoji: "✍️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "text"]
published: true
---

## この記事で学べること

Composeの**テキスト装飾**（AnnotatedString、SpanStyle、ClickableText）を解説します。

---

## 基本のテキストスタイル

```kotlin
@Composable
fun TextStyles() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("Display Large", style = MaterialTheme.typography.displayLarge)
        Text("Headline Medium", style = MaterialTheme.typography.headlineMedium)
        Text("Title Large", style = MaterialTheme.typography.titleLarge)
        Text("Body Medium", style = MaterialTheme.typography.bodyMedium)
        Text("Label Small", style = MaterialTheme.typography.labelSmall)

        // カスタムスタイル
        Text(
            "カスタム",
            fontSize = 20.sp,
            fontWeight = FontWeight.Bold,
            fontStyle = FontStyle.Italic,
            letterSpacing = 2.sp,
            lineHeight = 28.sp,
            textDecoration = TextDecoration.Underline,
            color = Color(0xFF6200EE)
        )
    }
}
```

---

## AnnotatedString: 部分装飾

```kotlin
@Composable
fun RichText() {
    val annotated = buildAnnotatedString {
        append("これは")

        withStyle(SpanStyle(fontWeight = FontWeight.Bold)) {
            append("太字")
        }

        append("と")

        withStyle(SpanStyle(color = Color.Red, fontSize = 20.sp)) {
            append("赤い大きな文字")
        }

        append("と")

        withStyle(
            SpanStyle(
                background = Color.Yellow,
                textDecoration = TextDecoration.Underline
            )
        ) {
            append("ハイライト付き下線")
        }

        append("のテキストです。")
    }

    Text(annotated)
}
```

---

## クリック可能テキスト

```kotlin
@Composable
fun ClickableRichText() {
    val uriHandler = LocalUriHandler.current

    val annotated = buildAnnotatedString {
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

    ClickableText(
        text = annotated,
        onClick = { offset ->
            annotated.getStringAnnotations(tag = "URL", start = offset, end = offset)
                .firstOrNull()?.let { annotation ->
                    uriHandler.openUri(annotation.item)
                }
        }
    )
}
```

---

## 段落スタイル

```kotlin
@Composable
fun ParagraphStyles() {
    val text = buildAnnotatedString {
        withStyle(ParagraphStyle(textAlign = TextAlign.Center)) {
            append("中央揃えのタイトル\n")
        }

        withStyle(
            ParagraphStyle(
                textIndent = TextIndent(firstLine = 24.sp),
                lineHeight = 24.sp
            )
        ) {
            append("本文テキストです。最初の行は字下げされています。")
            append("2行目以降は通常のインデントです。")
        }

        withStyle(ParagraphStyle(textAlign = TextAlign.End)) {
            append("\n— 著者名")
        }
    }

    Text(text)
}
```

---

## テキスト選択

```kotlin
@Composable
fun SelectableTextExample() {
    // 選択可能テキスト
    SelectionContainer {
        Column {
            Text("このテキストは選択可能です")
            Text("こちらも選択できます")

            // 部分的に選択不可
            DisableSelection {
                Text("このテキストは選択できません")
            }

            Text("ここはまた選択可能")
        }
    }
}
```

---

## テキストオーバーフロー

```kotlin
@Composable
fun TextOverflow() {
    // 省略記号
    Text(
        "とても長いテキストがここに入ります。画面幅を超えると省略されます。",
        maxLines = 1,
        overflow = TextOverflow.Ellipsis
    )

    // 展開/折りたたみ
    var expanded by remember { mutableStateOf(false) }
    Column {
        Text(
            "長い説明文がここに入ります。" +
            "詳細な内容が複数行にわたって記載されています。" +
            "ユーザーが展開ボタンを押すと全文が表示されます。",
            maxLines = if (expanded) Int.MAX_VALUE else 2,
            overflow = TextOverflow.Ellipsis
        )
        TextButton(onClick = { expanded = !expanded }) {
            Text(if (expanded) "折りたたむ" else "もっと見る")
        }
    }
}
```

---

## まとめ

- `MaterialTheme.typography`で統一スタイル
- `buildAnnotatedString` + `SpanStyle`で部分装飾
- `ClickableText` + `pushStringAnnotation`でリンク
- `ParagraphStyle`で段落レベルの装飾
- `SelectionContainer`/`DisableSelection`で選択制御
- `maxLines` + `TextOverflow.Ellipsis`で省略表示

---

8種類のAndroidアプリテンプレート（テキストUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Modifier完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-modifier-guide-2026)
- [カスタムテーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-theme-custom-2026)
- [入力フォーム完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-forms-input-2026)
