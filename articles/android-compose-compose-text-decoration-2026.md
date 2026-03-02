---
title: "TextDecoration完全ガイド — 下線/取り消し線/AnnotatedString装飾"
emoji: "✨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "text"]
published: true
---

## この記事で学べること

**TextDecoration**（下線、取り消し線、AnnotatedStringによる部分装飾）を解説します。

---

## TextDecoration基本

```kotlin
@Composable
fun TextDecorationExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Text(
            "下線テキスト",
            textDecoration = TextDecoration.Underline,
            fontSize = 18.sp
        )
        Text(
            "取り消し線テキスト",
            textDecoration = TextDecoration.LineThrough,
            fontSize = 18.sp
        )
        Text(
            "下線+取り消し線",
            textDecoration = TextDecoration.Underline + TextDecoration.LineThrough,
            fontSize = 18.sp
        )
    }
}
```

---

## AnnotatedString装飾

```kotlin
@Composable
fun RichTextExample() {
    val annotatedText = buildAnnotatedString {
        append("これは")
        withStyle(SpanStyle(fontWeight = FontWeight.Bold)) { append("太字") }
        append("と")
        withStyle(SpanStyle(color = Color.Red, fontSize = 20.sp)) { append("赤い大きな文字") }
        append("と")
        withStyle(SpanStyle(textDecoration = TextDecoration.Underline, color = Color.Blue)) {
            pushStringAnnotation("URL", "https://example.com")
            append("リンク風テキスト")
            pop()
        }
        append("の例です。")
    }

    ClickableText(
        text = annotatedText,
        onClick = { offset ->
            annotatedText.getStringAnnotations("URL", offset, offset)
                .firstOrNull()?.let { /* URLを開く */ }
        },
        style = TextStyle(fontSize = 16.sp)
    )
}
```

---

## タスクリスト表示

```kotlin
@Composable
fun TaskListItem(title: String, isCompleted: Boolean, onToggle: () -> Unit) {
    Row(
        Modifier
            .fillMaxWidth()
            .clickable(onClick = onToggle)
            .padding(12.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Checkbox(checked = isCompleted, onCheckedChange = { onToggle() })
        Spacer(Modifier.width(8.dp))
        Text(
            title,
            textDecoration = if (isCompleted) TextDecoration.LineThrough else TextDecoration.None,
            color = if (isCompleted) Color.Gray else Color.Unspecified,
            fontSize = 16.sp
        )
    }
}

@Composable
fun TaskListDemo() {
    val tasks = remember {
        mutableStateListOf(
            "買い物に行く" to true,
            "レポートを書く" to false,
            "メール返信" to false
        )
    }
    LazyColumn {
        itemsIndexed(tasks) { index, (title, done) ->
            TaskListItem(title, done) {
                tasks[index] = title to !done
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `TextDecoration.Underline` | 下線 |
| `TextDecoration.LineThrough` | 取り消し線 |
| `SpanStyle` | 部分スタイル |
| `buildAnnotatedString` | リッチテキスト |

- `TextDecoration`で下線・取り消し線を追加
- `+`演算子で複数のDecorationを組み合わせ
- `AnnotatedString`で部分的なスタイル適用
- タスク完了表示に`LineThrough`が最適

---

8種類のAndroidアプリテンプレート（タスク管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキストスタイル](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
- [Brush](https://zenn.dev/myougatheaxo/articles/android-compose-compose-brush-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
