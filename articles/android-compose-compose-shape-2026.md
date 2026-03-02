---
title: "Shape完全ガイド — カスタムShape/GenericShape/クリッピング/波形"
emoji: "🔷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Shape**（カスタムShape、GenericShape、クリッピング、波形/ダイヤモンド/星型）を解説します。

---

## 基本Shape

```kotlin
@Composable
fun ShapeExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Box(Modifier.size(100.dp).clip(RoundedCornerShape(16.dp)).background(Color(0xFF6200EE)))
        Box(Modifier.size(100.dp).clip(CircleShape).background(Color(0xFF03DAC5)))
        Box(Modifier.size(100.dp).clip(CutCornerShape(16.dp)).background(Color(0xFFFF5722)))
        Box(Modifier.size(100.dp).clip(RoundedCornerShape(topStart = 32.dp, bottomEnd = 32.dp)).background(Color(0xFF4CAF50)))
    }
}
```

---

## GenericShape（カスタム）

```kotlin
val DiamondShape = GenericShape { size, _ ->
    moveTo(size.width / 2f, 0f)
    lineTo(size.width, size.height / 2f)
    lineTo(size.width / 2f, size.height)
    lineTo(0f, size.height / 2f)
    close()
}

val StarShape = GenericShape { size, _ ->
    val cx = size.width / 2f
    val cy = size.height / 2f
    val outerRadius = minOf(cx, cy)
    val innerRadius = outerRadius * 0.4f
    val points = 5

    for (i in 0 until points * 2) {
        val radius = if (i % 2 == 0) outerRadius else innerRadius
        val angle = Math.PI / points * i - Math.PI / 2
        val x = cx + (radius * cos(angle)).toFloat()
        val y = cy + (radius * sin(angle)).toFloat()
        if (i == 0) moveTo(x, y) else lineTo(x, y)
    }
    close()
}

val WaveShape = GenericShape { size, _ ->
    moveTo(0f, size.height * 0.7f)
    val waveWidth = size.width / 4
    for (i in 0..3) {
        val x1 = waveWidth * i + waveWidth / 2
        val x2 = waveWidth * (i + 1)
        quadraticBezierTo(x1, size.height * (if (i % 2 == 0) 0.5f else 0.9f), x2, size.height * 0.7f)
    }
    lineTo(size.width, size.height)
    lineTo(0f, size.height)
    close()
}

// 使用
@Composable
fun CustomShapeDemo() {
    Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
        Box(Modifier.size(80.dp).clip(DiamondShape).background(Color(0xFFE91E63)))
        Box(Modifier.size(80.dp).clip(StarShape).background(Color(0xFFFFD700)))
        Box(Modifier.size(200.dp, 100.dp).clip(WaveShape).background(Color(0xFF2196F3)))
    }
}
```

---

## Shape付きSurface

```kotlin
@Composable
fun ShapedCard(shape: Shape) {
    Surface(
        shape = shape,
        tonalElevation = 4.dp,
        shadowElevation = 4.dp,
        modifier = Modifier.size(120.dp)
    ) {
        Box(contentAlignment = Alignment.Center) {
            Text("Card")
        }
    }
}
```

---

## まとめ

| Shape | 用途 |
|-------|------|
| RoundedCornerShape | 角丸 |
| CircleShape | 円形 |
| CutCornerShape | 切り欠き |
| GenericShape | カスタム形状 |

- `clip(Shape)`でView/Surfaceの形状をカスタマイズ
- `GenericShape`でPath操作による自由形状
- `quadraticBezierTo`で曲線形状（波形等）
- `Surface(shape = ...)`でMaterial3カード

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [Blur/Glassmorphism](https://zenn.dev/myougatheaxo/articles/android-compose-blur-glassmorphism-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
