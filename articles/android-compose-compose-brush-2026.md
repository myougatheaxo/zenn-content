---
title: "Brush完全ガイド — グラデーション/ShaderBrush/テキストブラシ"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "graphics"]
published: true
---

## この記事で学べること

**Brush**（linearGradient、radialGradient、sweepGradient、ShaderBrush、テキストへの適用）を解説します。

---

## Brush種類

```kotlin
@Composable
fun BrushExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // Linear Gradient
        Box(
            Modifier
                .fillMaxWidth()
                .height(60.dp)
                .background(
                    Brush.linearGradient(listOf(Color(0xFF42A5F5), Color(0xFF7E57C2)))
                )
        )

        // Radial Gradient
        Box(
            Modifier
                .size(120.dp)
                .background(
                    Brush.radialGradient(
                        listOf(Color.Yellow, Color.Red, Color.Transparent),
                        radius = 200f
                    )
                )
        )

        // Sweep Gradient
        Box(
            Modifier
                .size(120.dp)
                .background(
                    Brush.sweepGradient(
                        listOf(Color.Red, Color.Yellow, Color.Green, Color.Cyan, Color.Blue, Color.Red)
                    ),
                    shape = CircleShape
                )
        )
    }
}
```

---

## テキストにBrush適用

```kotlin
@Composable
fun GradientText() {
    val gradientBrush = Brush.linearGradient(
        listOf(Color(0xFFFF6B6B), Color(0xFF4ECDC4), Color(0xFF45B7D1))
    )

    Text(
        "グラデーションテキスト",
        style = TextStyle(
            brush = gradientBrush,
            fontSize = 32.sp,
            fontWeight = FontWeight.Bold
        )
    )
}

// アニメーション付きグラデーション
@Composable
fun AnimatedGradientText() {
    val infiniteTransition = rememberInfiniteTransition(label = "text")
    val offset by infiniteTransition.animateFloat(
        initialValue = 0f, targetValue = 2000f,
        animationSpec = infiniteRepeatable(tween(3000, easing = LinearEasing)),
        label = "offset"
    )

    Text(
        "アニメーション",
        style = TextStyle(
            brush = Brush.linearGradient(
                listOf(Color.Magenta, Color.Cyan, Color.Magenta),
                start = Offset(offset, 0f),
                end = Offset(offset + 500f, 0f)
            ),
            fontSize = 28.sp,
            fontWeight = FontWeight.Bold
        )
    )
}
```

---

## Canvas描画でBrush

```kotlin
@Composable
fun BrushCanvasDemo() {
    Canvas(Modifier.fillMaxWidth().height(200.dp)) {
        // グラデーション付き円
        drawCircle(
            brush = Brush.radialGradient(
                listOf(Color(0xFFFFD54F), Color(0xFFFF6F00)),
                center = center,
                radius = size.minDimension / 3
            ),
            radius = size.minDimension / 3
        )

        // グラデーション線
        drawLine(
            brush = Brush.horizontalGradient(
                listOf(Color.Red, Color.Blue)
            ),
            start = Offset(0f, size.height - 20f),
            end = Offset(size.width, size.height - 20f),
            strokeWidth = 4.dp.toPx()
        )
    }
}
```

---

## まとめ

| Brush | 用途 |
|-------|------|
| `linearGradient` | 直線グラデーション |
| `radialGradient` | 放射状グラデーション |
| `sweepGradient` | 円形グラデーション |
| `TextStyle.brush` | テキストに適用 |

- `Brush`はbackground/Canvas/TextStyleで使用可能
- `linearGradient`のstart/endで方向制御
- テキストの`brush`プロパティでグラデーション文字
- アニメーションと組み合わせて動的エフェクト

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [グラデーション](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gradient-2026)
- [drawWithCache](https://zenn.dev/myougatheaxo/articles/android-compose-compose-draw-with-cache-2026)
- [テキストスタイル](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-style-2026)
