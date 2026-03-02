---
title: "ComposeのCanvas描画ガイド — drawLine/drawArc/Path"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

Composeの**Canvas**での描画（基本図形、Path、アニメーション描画）を解説します。

---

## 基本図形

```kotlin
@Composable
fun BasicShapes() {
    Canvas(modifier = Modifier.size(300.dp)) {
        // 四角形
        drawRect(
            color = Color.Blue,
            topLeft = Offset(20f, 20f),
            size = Size(100f, 80f)
        )

        // 円
        drawCircle(
            color = Color.Red,
            radius = 50f,
            center = Offset(200f, 60f)
        )

        // 線
        drawLine(
            color = Color.Green,
            start = Offset(0f, 150f),
            end = Offset(size.width, 150f),
            strokeWidth = 4f
        )

        // 角丸四角形
        drawRoundRect(
            color = Color.Magenta,
            topLeft = Offset(20f, 180f),
            size = Size(200f, 60f),
            cornerRadius = CornerRadius(16f)
        )
    }
}
```

---

## Arc（円弧）

```kotlin
@Composable
fun ProgressArc(progress: Float) {
    Canvas(modifier = Modifier.size(200.dp)) {
        // 背景の円弧
        drawArc(
            color = Color.LightGray,
            startAngle = -90f,
            sweepAngle = 360f,
            useCenter = false,
            style = Stroke(width = 12f, cap = StrokeCap.Round)
        )

        // 進捗の円弧
        drawArc(
            color = Color(0xFF6200EA),
            startAngle = -90f,
            sweepAngle = 360f * progress,
            useCenter = false,
            style = Stroke(width = 12f, cap = StrokeCap.Round)
        )

        // 中央テキスト
        drawContext.canvas.nativeCanvas.drawText(
            "${(progress * 100).toInt()}%",
            size.width / 2,
            size.height / 2 + 16f,
            android.graphics.Paint().apply {
                textAlign = android.graphics.Paint.Align.CENTER
                textSize = 48f
                color = android.graphics.Color.BLACK
            }
        )
    }
}
```

---

## Path

```kotlin
@Composable
fun StarShape() {
    Canvas(modifier = Modifier.size(200.dp)) {
        val path = Path().apply {
            val cx = size.width / 2
            val cy = size.height / 2
            val outerRadius = size.minDimension / 2
            val innerRadius = outerRadius * 0.4f

            for (i in 0 until 5) {
                val outerAngle = Math.toRadians((i * 72 - 90).toDouble())
                val innerAngle = Math.toRadians((i * 72 + 36 - 90).toDouble())

                if (i == 0) {
                    moveTo(
                        cx + (outerRadius * cos(outerAngle)).toFloat(),
                        cy + (outerRadius * sin(outerAngle)).toFloat()
                    )
                } else {
                    lineTo(
                        cx + (outerRadius * cos(outerAngle)).toFloat(),
                        cy + (outerRadius * sin(outerAngle)).toFloat()
                    )
                }
                lineTo(
                    cx + (innerRadius * cos(innerAngle)).toFloat(),
                    cy + (innerRadius * sin(innerAngle)).toFloat()
                )
            }
            close()
        }

        drawPath(path, Color(0xFFFFD700))
        drawPath(path, Color(0xFFDAA520), style = Stroke(width = 3f))
    }
}
```

---

## アニメーション付き描画

```kotlin
@Composable
fun AnimatedCircle() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")
    val radius by infiniteTransition.animateFloat(
        initialValue = 50f,
        targetValue = 100f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "radius"
    )
    val alpha by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 0.3f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Canvas(modifier = Modifier.size(200.dp)) {
        drawCircle(
            color = Color(0xFF6200EA).copy(alpha = alpha),
            radius = radius,
            center = center
        )
    }
}
```

---

## まとめ

- `Canvas`でカスタム描画（drawRect/drawCircle/drawLine）
- `drawArc`で円弧/プログレスリング
- `Path`で星形等の複雑な図形
- `Stroke`でアウトライン描画、`StrokeCap.Round`で丸い線端
- `nativeCanvas`でテキスト描画
- `infiniteTransition`でアニメーション付き描画

---

8種類のAndroidアプリテンプレート（カスタムUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [グラデーション/Brush](https://zenn.dev/myougatheaxo/articles/android-compose-gradient-brush-2026)
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
- [カスタムレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
