---
title: "Compose Canvas完全ガイド — カスタム描画・グラフ・アニメーション"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "canvas"]
published: true
---

## この記事で学べること

Compose の **Canvas** を使ったカスタム描画の基本から、グラフやアニメーションまで解説します。

---

## 基本のCanvas

```kotlin
@Composable
fun BasicCanvas() {
    Canvas(modifier = Modifier.fillMaxWidth().height(200.dp)) {
        // 円
        drawCircle(
            color = Color.Blue,
            radius = 80f,
            center = center
        )

        // 四角
        drawRect(
            color = Color.Red.copy(alpha = 0.5f),
            topLeft = Offset(50f, 50f),
            size = Size(200f, 100f)
        )

        // 線
        drawLine(
            color = Color.Green,
            start = Offset(0f, size.height),
            end = Offset(size.width, 0f),
            strokeWidth = 4f
        )
    }
}
```

---

## 円弧・扇形

```kotlin
Canvas(modifier = Modifier.size(200.dp)) {
    // 円弧
    drawArc(
        color = Color.Blue,
        startAngle = 0f,
        sweepAngle = 270f,
        useCenter = false,
        style = Stroke(width = 8f, cap = StrokeCap.Round),
        size = Size(size.width, size.height)
    )

    // 扇形（useCenter = true）
    drawArc(
        color = Color.Red.copy(alpha = 0.3f),
        startAngle = 0f,
        sweepAngle = 90f,
        useCenter = true
    )
}
```

---

## Path（自由な図形）

```kotlin
Canvas(modifier = Modifier.size(200.dp)) {
    val path = Path().apply {
        moveTo(size.width / 2, 0f)
        lineTo(size.width, size.height)
        lineTo(0f, size.height)
        close()
    }

    drawPath(
        path = path,
        color = Color.Magenta,
        style = Stroke(width = 4f)
    )
}
```

---

## テキスト描画

```kotlin
@Composable
fun TextOnCanvas() {
    val textMeasurer = rememberTextMeasurer()

    Canvas(modifier = Modifier.fillMaxWidth().height(100.dp)) {
        drawText(
            textMeasurer = textMeasurer,
            text = "Canvas上のテキスト",
            topLeft = Offset(20f, 20f),
            style = TextStyle(
                fontSize = 24.sp,
                color = Color.Black,
                fontWeight = FontWeight.Bold
            )
        )
    }
}
```

---

## 棒グラフ

```kotlin
@Composable
fun BarChart(data: List<Float>, modifier: Modifier = Modifier) {
    Canvas(modifier = modifier.fillMaxWidth().height(200.dp)) {
        val barWidth = size.width / (data.size * 2)
        val maxValue = data.max()

        data.forEachIndexed { index, value ->
            val barHeight = (value / maxValue) * size.height
            val x = barWidth * (index * 2 + 0.5f)

            drawRect(
                color = Color(0xFF6200EE),
                topLeft = Offset(x, size.height - barHeight),
                size = Size(barWidth, barHeight),
                style = Fill
            )
        }
    }
}
```

---

## 折れ線グラフ

```kotlin
@Composable
fun LineChart(data: List<Float>, modifier: Modifier = Modifier) {
    Canvas(modifier = modifier.fillMaxWidth().height(200.dp)) {
        if (data.size < 2) return@Canvas
        val maxValue = data.max()
        val stepX = size.width / (data.size - 1)

        val path = Path().apply {
            data.forEachIndexed { index, value ->
                val x = index * stepX
                val y = size.height - (value / maxValue) * size.height
                if (index == 0) moveTo(x, y) else lineTo(x, y)
            }
        }

        drawPath(path, Color(0xFF03DAC5), style = Stroke(width = 4f, cap = StrokeCap.Round))

        // データポイント
        data.forEachIndexed { index, value ->
            val x = index * stepX
            val y = size.height - (value / maxValue) * size.height
            drawCircle(Color(0xFF03DAC5), radius = 6f, center = Offset(x, y))
        }
    }
}
```

---

## 円グラフ

```kotlin
@Composable
fun PieChart(slices: List<Pair<Float, Color>>, modifier: Modifier = Modifier) {
    Canvas(modifier = modifier.size(200.dp)) {
        val total = slices.sumOf { it.first.toDouble() }.toFloat()
        var startAngle = -90f

        slices.forEach { (value, color) ->
            val sweepAngle = (value / total) * 360f
            drawArc(
                color = color,
                startAngle = startAngle,
                sweepAngle = sweepAngle,
                useCenter = true
            )
            startAngle += sweepAngle
        }
    }
}
```

---

## アニメーション付きCanvas

```kotlin
@Composable
fun AnimatedProgress(progress: Float) {
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(durationMillis = 1000)
    )

    Canvas(modifier = Modifier.size(120.dp)) {
        // 背景の円
        drawArc(
            color = Color.LightGray,
            startAngle = 0f,
            sweepAngle = 360f,
            useCenter = false,
            style = Stroke(width = 12f, cap = StrokeCap.Round)
        )
        // 進捗
        drawArc(
            color = Color(0xFF6200EE),
            startAngle = -90f,
            sweepAngle = animatedProgress * 360f,
            useCenter = false,
            style = Stroke(width = 12f, cap = StrokeCap.Round)
        )
    }
}
```

---

## まとめ

- `Canvas` + `DrawScope`でカスタム描画
- `drawCircle`/`drawRect`/`drawLine`/`drawArc`/`drawPath`
- `rememberTextMeasurer`でテキスト描画
- 棒・折れ線・円グラフを自作可能
- `animateFloatAsState`でアニメーション付き描画

---

8種類のAndroidアプリテンプレート（カスタムUI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
