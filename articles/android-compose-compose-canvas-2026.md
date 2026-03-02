---
title: "Compose Canvas完全ガイド — drawLine/drawCircle/drawArc/カスタム描画"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

**Compose Canvas**（drawLine、drawCircle、drawArc、Path、カスタムグラフ描画）を解説します。

---

## 基本Canvas

```kotlin
@Composable
fun BasicCanvasDemo() {
    Canvas(modifier = Modifier.fillMaxWidth().height(200.dp)) {
        // 線
        drawLine(
            color = Color.Blue,
            start = Offset(0f, size.height / 2),
            end = Offset(size.width, size.height / 2),
            strokeWidth = 4f
        )

        // 円
        drawCircle(
            color = Color.Red,
            radius = 50f,
            center = center
        )

        // 矩形
        drawRect(
            color = Color.Green.copy(alpha = 0.3f),
            topLeft = Offset(20f, 20f),
            size = Size(100f, 80f)
        )
    }
}
```

---

## 円グラフ

```kotlin
@Composable
fun PieChart(data: List<Pair<String, Float>>, modifier: Modifier = Modifier) {
    val colors = listOf(Color(0xFF4CAF50), Color(0xFF2196F3), Color(0xFFFF9800), Color(0xFFE91E63))
    val total = data.sumOf { it.second.toDouble() }.toFloat()

    Canvas(modifier = modifier.size(200.dp)) {
        var startAngle = -90f
        data.forEachIndexed { index, (_, value) ->
            val sweep = (value / total) * 360f
            drawArc(
                color = colors[index % colors.size],
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
```

---

## 折れ線グラフ

```kotlin
@Composable
fun LineChart(points: List<Float>, modifier: Modifier = Modifier) {
    Canvas(modifier = modifier.fillMaxWidth().height(150.dp)) {
        if (points.size < 2) return@Canvas
        val maxVal = points.max()
        val minVal = points.min()
        val range = (maxVal - minVal).coerceAtLeast(1f)
        val stepX = size.width / (points.size - 1)

        val path = Path().apply {
            points.forEachIndexed { i, value ->
                val x = i * stepX
                val y = size.height - ((value - minVal) / range) * size.height
                if (i == 0) moveTo(x, y) else lineTo(x, y)
            }
        }

        drawPath(path, Color(0xFF2196F3), style = Stroke(width = 3f, cap = StrokeCap.Round))

        // データポイント
        points.forEachIndexed { i, value ->
            val x = i * stepX
            val y = size.height - ((value - minVal) / range) * size.height
            drawCircle(Color(0xFF2196F3), radius = 5f, center = Offset(x, y))
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `drawLine` | 直線描画 |
| `drawCircle` | 円描画 |
| `drawArc` | 扇/弧描画 |
| `drawPath` | パス描画 |

- `Canvas`で自由な2D描画が可能
- `drawArc`+`useCenter`で円グラフ
- `Path`+`drawPath`で折れ線グラフ
- `size`/`center`でキャンバスサイズ参照

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Path](https://zenn.dev/myougatheaxo/articles/android-compose-compose-path-2026)
- [Compose Brush](https://zenn.dev/myougatheaxo/articles/android-compose-compose-brush-2026)
- [Compose BlendMode](https://zenn.dev/myougatheaxo/articles/android-compose-compose-blend-mode-2026)
