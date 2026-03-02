---
title: "レスポンシブテキストガイド — autoSize/maxLines/overflow処理"
emoji: "📏"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "text"]
published: true
---

## この記事で学べること

Composeでの**テキストのレスポンシブ表示**（自動サイズ調整、行数制御、オーバーフロー処理）を解説します。

---

## テキストのオーバーフロー

```kotlin
@Composable
fun TextOverflowExamples() {
    Column(Modifier.padding(16.dp)) {
        // 省略表示
        Text(
            "これは非常に長いテキストで、1行に収まりません。省略記号で表示されます。",
            maxLines = 1,
            overflow = TextOverflow.Ellipsis
        )

        Spacer(Modifier.height(8.dp))

        // 2行制限
        Text(
            "長いテキストを2行まで表示します。それ以上は省略されます。3行目以降はここで切られます。",
            maxLines = 2,
            overflow = TextOverflow.Ellipsis
        )

        Spacer(Modifier.height(8.dp))

        // クリップ
        Text(
            "クリップされるテキスト",
            maxLines = 1,
            overflow = TextOverflow.Clip
        )
    }
}
```

---

## 展開/折りたたみテキスト

```kotlin
@Composable
fun ExpandableText(
    text: String,
    collapsedMaxLines: Int = 3
) {
    var expanded by remember { mutableStateOf(false) }
    var hasOverflow by remember { mutableStateOf(false) }

    Column {
        Text(
            text = text,
            maxLines = if (expanded) Int.MAX_VALUE else collapsedMaxLines,
            overflow = TextOverflow.Ellipsis,
            onTextLayout = { result ->
                hasOverflow = result.hasVisualOverflow
            }
        )

        if (hasOverflow || expanded) {
            TextButton(onClick = { expanded = !expanded }) {
                Text(if (expanded) "閉じる" else "もっと見る")
            }
        }
    }
}
```

---

## 自動フォントサイズ調整

```kotlin
@Composable
fun AutoSizeText(
    text: String,
    modifier: Modifier = Modifier,
    maxFontSize: TextUnit = 24.sp,
    minFontSize: TextUnit = 12.sp,
    maxLines: Int = 1
) {
    var fontSize by remember { mutableStateOf(maxFontSize) }

    Text(
        text = text,
        fontSize = fontSize,
        maxLines = maxLines,
        overflow = TextOverflow.Ellipsis,
        onTextLayout = { result ->
            if (result.hasVisualOverflow && fontSize > minFontSize) {
                fontSize = (fontSize.value - 1).sp
            }
        },
        modifier = modifier
    )
}

// 使用例
AutoSizeText(
    text = "この文字は領域に合わせて縮小します",
    modifier = Modifier.fillMaxWidth(),
    maxFontSize = 32.sp,
    minFontSize = 14.sp
)
```

---

## 行数に応じたスタイル変更

```kotlin
@Composable
fun AdaptiveTitle(title: String) {
    var lineCount by remember { mutableIntStateOf(1) }

    Text(
        text = title,
        style = if (lineCount > 1) {
            MaterialTheme.typography.titleMedium
        } else {
            MaterialTheme.typography.titleLarge
        },
        maxLines = 2,
        overflow = TextOverflow.Ellipsis,
        onTextLayout = { result ->
            lineCount = result.lineCount
        }
    )
}
```

---

## 長い数値の表示

```kotlin
@Composable
fun CompactNumber(count: Long) {
    val text = when {
        count >= 1_000_000 -> "%.1fM".format(count / 1_000_000.0)
        count >= 1_000 -> "%.1fK".format(count / 1_000.0)
        else -> count.toString()
    }
    Text(text, style = MaterialTheme.typography.labelMedium)
}

// 使用
CompactNumber(1234)     // 1.2K
CompactNumber(1234567)  // 1.2M
```

---

## まとめ

- `maxLines` + `overflow = TextOverflow.Ellipsis`で省略表示
- `onTextLayout`で行数やオーバーフロー検出
- 展開/折りたたみで長文テキスト対応
- `fontSize`の動的変更で自動サイズ調整
- 行数に応じたスタイル切り替え
- CompactNumberで長い数値を短縮表示

---

8種類のAndroidアプリテンプレート（テキスト表示最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テキスト装飾ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-text-styling-2026)
- [Typography/フォントガイド](https://zenn.dev/myougatheaxo/articles/android-compose-typography-font-2026)
- [レスポンシブレイアウトガイド](https://zenn.dev/myougatheaxo/articles/android-compose-multi-screen-2026)
