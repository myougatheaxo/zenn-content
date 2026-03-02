---
title: "Compose Shadow完全ガイド — elevation/shadow/drawBehind影/カスタムシャドウ"
emoji: "🌑"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Compose Shadow**（elevation、Modifier.shadow、drawBehind影、カスタムシャドウ効果）を解説します。

---

## 基本Shadow

```kotlin
@Composable
fun ShadowDemo() {
    Column(Modifier.padding(24.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // elevation
        Card(elevation = CardDefaults.cardElevation(defaultElevation = 8.dp)) {
            Text("Elevation 8dp", Modifier.padding(16.dp))
        }

        // Modifier.shadow
        Box(
            Modifier
                .shadow(elevation = 12.dp, shape = RoundedCornerShape(16.dp))
                .background(Color.White, RoundedCornerShape(16.dp))
                .padding(16.dp)
        ) {
            Text("Shadow 12dp")
        }

        // 円形シャドウ
        Box(
            Modifier
                .size(80.dp)
                .shadow(8.dp, CircleShape)
                .background(Color.White, CircleShape),
            contentAlignment = Alignment.Center
        ) {
            Text("円")
        }
    }
}
```

---

## カスタムカラーシャドウ

```kotlin
fun Modifier.coloredShadow(
    color: Color = Color.Black,
    alpha: Float = 0.2f,
    borderRadius: Dp = 0.dp,
    shadowRadius: Dp = 8.dp,
    offsetX: Dp = 0.dp,
    offsetY: Dp = 4.dp
) = this.drawBehind {
    val shadowColor = color.copy(alpha = alpha).toArgb()
    val transparent = color.copy(alpha = 0f).toArgb()
    drawIntoCanvas { canvas ->
        val paint = Paint().apply {
            asFrameworkPaint().apply {
                this.color = transparent
                setShadowLayer(
                    shadowRadius.toPx(), offsetX.toPx(), offsetY.toPx(), shadowColor
                )
            }
        }
        canvas.drawRoundRect(
            0f, 0f, size.width, size.height,
            borderRadius.toPx(), borderRadius.toPx(), paint
        )
    }
}

@Composable
fun ColoredShadowDemo() {
    Box(
        Modifier
            .coloredShadow(
                color = Color.Blue,
                alpha = 0.3f,
                borderRadius = 16.dp,
                shadowRadius = 12.dp
            )
            .background(Color.White, RoundedCornerShape(16.dp))
            .padding(24.dp)
    ) {
        Text("ブルーシャドウ", style = MaterialTheme.typography.titleMedium)
    }
}
```

---

## アニメーションシャドウ

```kotlin
@Composable
fun AnimatedShadowCard(isPressed: Boolean) {
    val elevation by animateDpAsState(
        targetValue = if (isPressed) 2.dp else 8.dp,
        animationSpec = tween(200), label = "elevation"
    )

    Card(
        elevation = CardDefaults.cardElevation(defaultElevation = elevation),
        modifier = Modifier
            .fillMaxWidth()
            .pointerInput(Unit) {
                detectTapGestures(
                    onPress = { /* handle */ }
                )
            }
    ) {
        Text("タップでシャドウ変化", Modifier.padding(16.dp))
    }
}
```

---

## まとめ

| 方法 | 用途 |
|------|------|
| `elevation` | Card標準シャドウ |
| `Modifier.shadow` | 任意Viewにシャドウ |
| `drawBehind` | カスタムカラーシャドウ |
| `animateDpAsState` | シャドウアニメーション |

- `Modifier.shadow`で任意のViewにシャドウ追加
- `drawBehind`+`setShadowLayer`でカラーシャドウ
- `shape`パラメータでシャドウ形状を指定
- アニメーションで動的なシャドウ変化

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
- [Compose CustomShape](https://zenn.dev/myougatheaxo/articles/android-compose-compose-custom-shape-2026)
- [Compose GraphicsLayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-graphicsLayer-2026)
