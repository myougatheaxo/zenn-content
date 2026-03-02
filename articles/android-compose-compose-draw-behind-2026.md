---
title: "drawBehind/drawWithContent完全ガイド — カスタム背景描画/オーバーレイ/装飾"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

**drawBehind/drawWithContent**（カスタム背景描画、オーバーレイ、バッジ、装飾描画）を解説します。

---

## drawBehind

```kotlin
@Composable
fun DrawBehindExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // カスタム背景
        Text(
            "カスタム背景",
            modifier = Modifier
                .drawBehind {
                    drawRoundRect(
                        color = Color(0xFFE3F2FD),
                        cornerRadius = CornerRadius(8.dp.toPx())
                    )
                }
                .padding(12.dp),
            fontSize = 18.sp
        )

        // 下線
        Text(
            "下線付きテキスト",
            modifier = Modifier
                .drawBehind {
                    val strokeWidth = 2.dp.toPx()
                    drawLine(
                        color = Color.Red,
                        start = Offset(0f, size.height),
                        end = Offset(size.width, size.height),
                        strokeWidth = strokeWidth
                    )
                }
                .padding(bottom = 4.dp),
            fontSize = 18.sp
        )
    }
}
```

---

## drawWithContent

```kotlin
@Composable
fun DrawWithContentExample() {
    // バッジオーバーレイ
    Box(
        Modifier
            .size(60.dp)
            .drawWithContent {
                drawContent()
                // コンテンツの上に赤いドット
                drawCircle(
                    color = Color.Red,
                    radius = 8.dp.toPx(),
                    center = Offset(size.width - 4.dp.toPx(), 4.dp.toPx())
                )
            }
    ) {
        Icon(Icons.Default.Notifications, null, Modifier.size(48.dp).align(Alignment.Center))
    }
}

// グラデーションオーバーレイ
@Composable
fun GradientOverlayImage() {
    AsyncImage(
        model = "https://example.com/photo.jpg",
        contentDescription = null,
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .drawWithContent {
                drawContent()
                drawRect(
                    brush = Brush.verticalGradient(
                        listOf(Color.Transparent, Color.Black.copy(alpha = 0.7f))
                    )
                )
            },
        contentScale = ContentScale.Crop
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `drawBehind` | コンテンツの背面に描画 |
| `drawWithContent` | 前面+背面に描画 |
| `drawContent()` | 元コンテンツの描画 |
| `drawRoundRect` | 角丸矩形 |

- `drawBehind`でカスタム背景描画
- `drawWithContent`で前面オーバーレイ描画
- `drawContent()`の呼び出し位置で前後を制御
- バッジ/グラデーション/装飾に活用

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [グラデーション](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gradient-2026)
- [カスタムModifier](https://zenn.dev/myougatheaxo/articles/android-compose-compose-custom-modifier-2026)
