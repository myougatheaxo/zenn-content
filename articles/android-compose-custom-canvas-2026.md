---
title: "Canvas描画完全ガイド — drawCircle/Path/グラデーション/チャート"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

**Canvas描画**（基本図形、Path、グラデーション、円グラフ、折れ線グラフ）を解説します。

---

## 基本図形

```kotlin
@Composable
fun BasicShapes() {
    Canvas(Modifier.size(300.dp)) {
        // 円
        drawCircle(
            color = Color(0xFF6200EE),
            radius = 50f,
            center = Offset(100f, 100f)
        )

        // 矩形
        drawRect(
            color = Color(0xFF03DAC5),
            topLeft = Offset(200f, 50f),
            size = Size(150f, 100f),
            style = Stroke(width = 4f)
        )

        // 線
        drawLine(
            color = Color.Red,
            start = Offset(0f, 250f),
            end = Offset(size.width, 250f),
            strokeWidth = 3f
        )

        // 角丸矩形
        drawRoundRect(
            color = Color(0xFFFF5722),
            topLeft = Offset(50f, 300f),
            size = Size(200f, 80f),
            cornerRadius = CornerRadius(16f)
        )
    }
}
```

---

## 円グラフ

```kotlin
@Composable
fun PieChart(data: List<PieData>, modifier: Modifier = Modifier) {
    Canvas(modifier.size(200.dp)) {
        val total = data.sumOf { it.value.toDouble() }.toFloat()
        var startAngle = -90f

        data.forEach { slice ->
            val sweep = (slice.value / total) * 360f
            drawArc(
                color = slice.color,
                startAngle = startAngle,
                sweepAngle = sweep,
                useCenter = true,
                size = Size(size.minDimension, size.minDimension),
                topLeft = Offset(
                    (size.width - size.minDimension) / 2,
                    (size.height - size.minDimension) / 2
                )
            )
            startAngle += sweep
        }
    }
}

data class PieData(val label: String, val value: Float, val color: Color)
```

---

## 折れ線グラフ

```kotlin
@Composable
fun LineChart(points: List<Float>, modifier: Modifier = Modifier) {
    Canvas(modifier.fillMaxWidth().height(200.dp)) {
        if (points.isEmpty()) return@Canvas

        val maxVal = points.max()
        val minVal = points.min()
        val range = maxVal - minVal
        val stepX = size.width / (points.size - 1)

        val path = Path()
        points.forEachIndexed { index, value ->
            val x = index * stepX
            val y = size.height - ((value - minVal) / range * size.height)

            if (index == 0) path.moveTo(x, y)
            else path.lineTo(x, y)
        }

        // グラデーション塗り
        val fillPath = Path().apply {
            addPath(path)
            lineTo(size.width, size.height)
            lineTo(0f, size.height)
            close()
        }
        drawPath(
            fillPath,
            brush = Brush.verticalGradient(
                colors = listOf(Color(0xFF6200EE).copy(alpha = 0.3f), Color.Transparent)
            )
        )

        // 線
        drawPath(path, color = Color(0xFF6200EE), style = Stroke(width = 3f, cap = StrokeCap.Round))

        // 点
        points.forEachIndexed { index, value ->
            val x = index * stepX
            val y = size.height - ((value - minVal) / range * size.height)
            drawCircle(Color(0xFF6200EE), radius = 5f, center = Offset(x, y))
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `drawCircle` | 円 |
| `drawRect` | 矩形 |
| `drawPath` | 自由曲線 |
| `drawArc` | 円弧/パイ |

- `Canvas`でカスタム描画を実現
- `Path`で折れ線グラフを描画
- `Brush.verticalGradient`でグラデーション
- `drawArc`で円グラフを作成

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
- [カスタムレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-layout-modifier-2026)
- [Blur/Glassmorphism](https://zenn.dev/myougatheaxo/articles/android-compose-blur-glassmorphism-2026)
