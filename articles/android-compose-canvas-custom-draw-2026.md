---
title: "Canvas/カスタム描画完全ガイド — drawCircle/Path/グラフ"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

**Canvas/カスタム描画**（drawCircle、Path、グラフ描画、タッチ操作、アニメーション）を解説します。

---

## 基本Canvas

```kotlin
@Composable
fun BasicCanvasExample() {
    Canvas(
        modifier = Modifier.fillMaxWidth().height(200.dp)
    ) {
        // 円
        drawCircle(
            color = Color.Blue,
            radius = 50f,
            center = Offset(100f, 100f)
        )

        // 矩形
        drawRect(
            color = Color.Red,
            topLeft = Offset(200f, 50f),
            size = Size(100f, 100f)
        )

        // 線
        drawLine(
            color = Color.Green,
            start = Offset(350f, 50f),
            end = Offset(450f, 150f),
            strokeWidth = 4f
        )

        // 角丸矩形
        drawRoundRect(
            color = Color.Magenta,
            topLeft = Offset(500f, 50f),
            size = Size(120f, 80f),
            cornerRadius = CornerRadius(16f)
        )
    }
}
```

---

## Pathで複雑な図形

```kotlin
@Composable
fun StarShape() {
    Canvas(Modifier.size(200.dp)) {
        val path = Path().apply {
            val cx = size.width / 2
            val cy = size.height / 2
            val outerRadius = size.minDimension / 2
            val innerRadius = outerRadius * 0.4f

            for (i in 0 until 5) {
                val outerAngle = Math.toRadians((i * 72.0 - 90))
                val innerAngle = Math.toRadians((i * 72.0 + 36 - 90))

                if (i == 0) {
                    moveTo(
                        cx + outerRadius * cos(outerAngle).toFloat(),
                        cy + outerRadius * sin(outerAngle).toFloat()
                    )
                } else {
                    lineTo(
                        cx + outerRadius * cos(outerAngle).toFloat(),
                        cy + outerRadius * sin(outerAngle).toFloat()
                    )
                }
                lineTo(
                    cx + innerRadius * cos(innerAngle).toFloat(),
                    cy + innerRadius * sin(innerAngle).toFloat()
                )
            }
            close()
        }

        drawPath(path, color = Color(0xFFFFD700))
    }
}
```

---

## 折れ線グラフ

```kotlin
@Composable
fun LineChart(
    dataPoints: List<Float>,
    modifier: Modifier = Modifier.fillMaxWidth().height(200.dp)
) {
    val primaryColor = MaterialTheme.colorScheme.primary

    Canvas(modifier) {
        if (dataPoints.isEmpty()) return@Canvas

        val maxValue = dataPoints.max()
        val minValue = dataPoints.min()
        val range = maxValue - minValue
        val stepX = size.width / (dataPoints.size - 1)
        val padding = 16f

        // グリッド線
        for (i in 0..4) {
            val y = padding + (size.height - padding * 2) * i / 4
            drawLine(Color.LightGray, Offset(0f, y), Offset(size.width, y), 1f)
        }

        // データ線
        val path = Path()
        dataPoints.forEachIndexed { index, value ->
            val x = index * stepX
            val y = padding + (size.height - padding * 2) * (1 - (value - minValue) / range)

            if (index == 0) path.moveTo(x, y) else path.lineTo(x, y)

            // ポイント
            drawCircle(primaryColor, 6f, Offset(x, y))
        }

        drawPath(path, primaryColor, style = Stroke(width = 3f, cap = StrokeCap.Round))

        // グラデーション塗りつぶし
        val fillPath = Path().apply {
            addPath(path)
            lineTo(size.width, size.height)
            lineTo(0f, size.height)
            close()
        }
        drawPath(
            fillPath,
            brush = Brush.verticalGradient(
                colors = listOf(primaryColor.copy(alpha = 0.3f), Color.Transparent)
            )
        )
    }
}

// 使用
LineChart(dataPoints = listOf(10f, 25f, 18f, 30f, 22f, 35f, 28f))
```

---

## タッチ操作付きCanvas

```kotlin
@Composable
fun DrawingCanvas() {
    val paths = remember { mutableStateListOf<Pair<Path, Color>>() }
    var currentPath by remember { mutableStateOf(Path()) }
    val currentColor = MaterialTheme.colorScheme.primary

    Canvas(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectDragGestures(
                    onDragStart = { offset ->
                        currentPath = Path().apply { moveTo(offset.x, offset.y) }
                    },
                    onDrag = { change, _ ->
                        currentPath.lineTo(change.position.x, change.position.y)
                    },
                    onDragEnd = {
                        paths.add(currentPath to currentColor)
                        currentPath = Path()
                    }
                )
            }
    ) {
        paths.forEach { (path, color) ->
            drawPath(path, color, style = Stroke(width = 4f, cap = StrokeCap.Round))
        }
        drawPath(currentPath, currentColor, style = Stroke(width = 4f, cap = StrokeCap.Round))
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `drawCircle` | 円描画 |
| `drawRect` | 矩形描画 |
| `drawLine` | 線描画 |
| `drawPath` | 複雑な図形 |
| `drawText` | テキスト描画 |
| `Brush` | グラデーション |

- `Canvas`コンポーザブルで自由描画
- `Path`で複雑な図形・グラフ
- `pointerInput`でタッチ操作
- `animateFloatAsState`でアニメーション

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
- [ConstraintLayout](https://zenn.dev/myougatheaxo/articles/android-compose-constraint-layout-compose-2026)
