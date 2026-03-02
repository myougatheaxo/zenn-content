---
title: "テキスト測定/レイアウト完全ガイド — TextMeasurer/onTextLayout/AutoSize"
emoji: "📝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "text"]
published: true
---

## この記事で学べること

**テキスト測定**（TextMeasurer、onTextLayout、AutoSizeText、テキスト省略、行数制御）を解説します。

---

## TextMeasurer

```kotlin
@Composable
fun MeasuredTextExample() {
    val textMeasurer = rememberTextMeasurer()
    val style = MaterialTheme.typography.bodyLarge

    Canvas(Modifier.fillMaxWidth().height(200.dp)) {
        val textResult = textMeasurer.measure(
            text = "Hello, Compose!",
            style = style,
            constraints = Constraints.fixedWidth(size.width.toInt())
        )

        drawText(
            textLayoutResult = textResult,
            topLeft = Offset(
                (size.width - textResult.size.width) / 2,
                (size.height - textResult.size.height) / 2
            )
        )
    }
}
```

---

## onTextLayout

```kotlin
@Composable
fun ExpandableText(text: String, maxLines: Int = 3) {
    var isExpanded by remember { mutableStateOf(false) }
    var hasOverflow by remember { mutableStateOf(false) }

    Column {
        Text(
            text = text,
            maxLines = if (isExpanded) Int.MAX_VALUE else maxLines,
            overflow = TextOverflow.Ellipsis,
            onTextLayout = { result ->
                hasOverflow = result.hasVisualOverflow
            }
        )
        if (hasOverflow || isExpanded) {
            TextButton(onClick = { isExpanded = !isExpanded }) {
                Text(if (isExpanded) "閉じる" else "もっと見る")
            }
        }
    }
}
```

---

## 自動サイズ調整テキスト

```kotlin
@Composable
fun AutoSizeText(
    text: String,
    modifier: Modifier = Modifier,
    maxFontSize: TextUnit = 24.sp,
    minFontSize: TextUnit = 10.sp,
    style: TextStyle = LocalTextStyle.current
) {
    var fontSize by remember { mutableStateOf(maxFontSize) }
    var readyToDraw by remember { mutableStateOf(false) }

    Text(
        text = text,
        style = style.copy(fontSize = fontSize),
        maxLines = 1,
        overflow = TextOverflow.Clip,
        modifier = modifier.drawWithContent {
            if (readyToDraw) drawContent()
        },
        onTextLayout = { result ->
            if (result.hasVisualOverflow && fontSize > minFontSize) {
                fontSize *= 0.9f
            } else {
                readyToDraw = true
            }
        }
    )
}
```

---

## AnnotatedString + テキスト内リンク

```kotlin
@Composable
fun LinkedText() {
    val annotatedString = buildAnnotatedString {
        append("利用規約に同意する場合は")
        pushStringAnnotation(tag = "URL", annotation = "https://example.com/terms")
        withStyle(SpanStyle(color = MaterialTheme.colorScheme.primary, textDecoration = TextDecoration.Underline)) {
            append("こちら")
        }
        pop()
        append("をご確認ください。")
    }

    ClickableText(
        text = annotatedString,
        onClick = { offset ->
            annotatedString.getStringAnnotations("URL", offset, offset).firstOrNull()?.let { annotation ->
                // URLを開く
            }
        }
    )
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| テキスト測定 | `rememberTextMeasurer()` |
| レイアウト情報 | `onTextLayout` |
| 省略検出 | `hasVisualOverflow` |
| Canvas描画 | `drawText()` |
| リンクテキスト | `AnnotatedString` |

- `TextMeasurer`でCanvas上にテキスト描画
- `onTextLayout`で省略・行数を検出
- `AutoSizeText`でテキストを自動縮小
- `AnnotatedString`でリッチテキスト表示

---

8種類のAndroidアプリテンプレート（カスタムテキスト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキストスタイリング](https://zenn.dev/myougatheaxo/articles/android-compose-text-styling-2026)
- [Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-custom-draw-2026)
- [Typography](https://zenn.dev/myougatheaxo/articles/android-compose-typography-font-2026)
