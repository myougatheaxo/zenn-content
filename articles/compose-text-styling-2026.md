---
title: "Composeテキスト完全ガイド — フォント・スタイル・AnnotatedStringの使い方"
emoji: "✏️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "typography"]
published: true
---

## この記事で学べること

ComposeのText表示を完全マスター。フォントサイズ、色、太字、複数スタイル混在、クリック可能テキストなど**全パターン**を解説します。

---

## 基本のText

```kotlin
Text(
    text = "Hello, Compose!",
    fontSize = 24.sp,
    fontWeight = FontWeight.Bold,
    color = MaterialTheme.colorScheme.primary
)
```

---

## Material3 Typography

```kotlin
Text("見出し", style = MaterialTheme.typography.headlineLarge)
Text("タイトル", style = MaterialTheme.typography.titleMedium)
Text("本文", style = MaterialTheme.typography.bodyLarge)
Text("ラベル", style = MaterialTheme.typography.labelSmall)
```

| スタイル | 用途 |
|---------|------|
| `displayLarge/Medium/Small` | 大見出し（ヒーロー） |
| `headlineLarge/Medium/Small` | セクション見出し |
| `titleLarge/Medium/Small` | カードタイトル |
| `bodyLarge/Medium/Small` | 本文 |
| `labelLarge/Medium/Small` | ボタン・タブ |

---

## テキスト装飾

```kotlin
// 下線
Text(
    text = "下線テキスト",
    textDecoration = TextDecoration.Underline
)

// 取り消し線
Text(
    text = "取り消し線",
    textDecoration = TextDecoration.LineThrough
)

// 斜体
Text(
    text = "イタリック",
    fontStyle = FontStyle.Italic
)

// 文字間隔
Text(
    text = "広い文字間隔",
    letterSpacing = 4.sp
)
```

---

## 複数行の制御

```kotlin
Text(
    text = "長いテキストが複数行にわたる場合の表示制御。overflow設定でテキストの切り詰め方を指定できます。",
    maxLines = 2,
    overflow = TextOverflow.Ellipsis,  // ... で省略
    lineHeight = 24.sp
)
```

| overflow | 効果 |
|---------|------|
| `Clip` | はみ出しを切り取り |
| `Ellipsis` | ... で省略 |
| `Visible` | はみ出して表示 |

---

## AnnotatedString（複数スタイル混在）

```kotlin
@Composable
fun StyledText() {
    val annotatedString = buildAnnotatedString {
        append("この文章には")

        withStyle(SpanStyle(
            fontWeight = FontWeight.Bold,
            color = MaterialTheme.colorScheme.primary
        )) {
            append("太字の部分")
        }

        append("と")

        withStyle(SpanStyle(
            fontSize = 20.sp,
            color = MaterialTheme.colorScheme.error
        )) {
            append("赤い大きい文字")
        }

        append("があります。")
    }

    Text(annotatedString)
}
```

---

## クリック可能テキスト

```kotlin
@Composable
fun ClickableTextExample() {
    val annotatedString = buildAnnotatedString {
        append("利用規約に同意するには")

        pushStringAnnotation(tag = "URL", annotation = "https://example.com/terms")
        withStyle(SpanStyle(
            color = MaterialTheme.colorScheme.primary,
            textDecoration = TextDecoration.Underline
        )) {
            append("こちら")
        }
        pop()

        append("をタップしてください。")
    }

    val context = LocalContext.current

    ClickableText(
        text = annotatedString,
        onClick = { offset ->
            annotatedString.getStringAnnotations("URL", offset, offset)
                .firstOrNull()?.let { annotation ->
                    val intent = Intent(Intent.ACTION_VIEW, Uri.parse(annotation.item))
                    context.startActivity(intent)
                }
        }
    )
}
```

---

## カスタムフォント

```kotlin
// res/font/ にフォントファイルを配置
val customFontFamily = FontFamily(
    Font(R.font.noto_sans_regular, FontWeight.Normal),
    Font(R.font.noto_sans_bold, FontWeight.Bold)
)

Text(
    text = "カスタムフォント",
    fontFamily = customFontFamily,
    fontWeight = FontWeight.Bold
)
```

---

## テキスト選択

```kotlin
SelectionContainer {
    Column {
        Text("このテキストは選択可能です")
        Text("こちらも選択できます")
    }
}
```

`SelectionContainer`で囲むとテキストのコピーが可能になります。

---

## まとめ

| 機能 | 方法 |
|------|------|
| 基本スタイル | `fontSize`, `fontWeight`, `color` |
| Material3準拠 | `MaterialTheme.typography.XXX` |
| 装飾 | `textDecoration`, `fontStyle` |
| 複数スタイル | `buildAnnotatedString` |
| リンク | `ClickableText` + `pushStringAnnotation` |
| カスタムフォント | `FontFamily` + `Font` |
| テキスト選択 | `SelectionContainer` |

---

8種類のAndroidアプリテンプレート（Material3 Typography設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [アクセシビリティ対応入門](https://zenn.dev/myougatheaxo/articles/android-accessibility-basics-2026)
