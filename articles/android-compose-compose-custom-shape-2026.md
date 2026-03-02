---
title: "CustomShape完全ガイド — カスタムシェイプ/GenericShape/Outline"
emoji: "🔷"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "graphics"]
published: true
---

## この記事で学べること

**CustomShape**（GenericShape、カスタムOutline、clip/background適用）を解説します。

---

## GenericShape

```kotlin
val TriangleShape = GenericShape { size, _ ->
    moveTo(size.width / 2, 0f)
    lineTo(size.width, size.height)
    lineTo(0f, size.height)
    close()
}

val DiamondShape = GenericShape { size, _ ->
    moveTo(size.width / 2, 0f)
    lineTo(size.width, size.height / 2)
    lineTo(size.width / 2, size.height)
    lineTo(0f, size.height / 2)
    close()
}

@Composable
fun CustomShapeDemo() {
    Row(Modifier.padding(16.dp), horizontalArrangement = Arrangement.spacedBy(16.dp)) {
        Box(Modifier.size(100.dp).clip(TriangleShape).background(Color(0xFF42A5F5)))
        Box(Modifier.size(100.dp).clip(DiamondShape).background(Color(0xFFE91E63)))
    }
}
```

---

## 波形・ハート

```kotlin
val WaveShape = GenericShape { size, _ ->
    moveTo(0f, size.height * 0.5f)
    val waveLength = size.width / 4
    for (i in 0..3) {
        quadraticBezierTo(
            waveLength * i + waveLength * 0.5f, size.height * 0.2f,
            waveLength * (i + 1), size.height * 0.5f
        )
    }
    lineTo(size.width, size.height)
    lineTo(0f, size.height)
    close()
}

val HeartShape = GenericShape { size, _ ->
    val w = size.width; val h = size.height
    moveTo(w / 2, h * 0.3f)
    cubicTo(w * 0.15f, -h * 0.1f, -w * 0.1f, h * 0.45f, w / 2, h)
    moveTo(w / 2, h * 0.3f)
    cubicTo(w * 0.85f, -h * 0.1f, w * 1.1f, h * 0.45f, w / 2, h)
}

@Composable
fun FancyShapes() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Box(Modifier.fillMaxWidth().height(100.dp).clip(WaveShape).background(Color(0xFF26C6DA)))
        Box(Modifier.size(120.dp).clip(HeartShape).background(Color.Red))
    }
}
```

---

## 画像クリッピング

```kotlin
val HexagonShape = GenericShape { size, _ ->
    val r = size.minDimension / 2
    val cx = size.width / 2; val cy = size.height / 2
    for (i in 0..5) {
        val angle = Math.toRadians((60.0 * i - 30))
        val x = cx + r * cos(angle).toFloat()
        val y = cy + r * sin(angle).toFloat()
        if (i == 0) moveTo(x, y) else lineTo(x, y)
    }
    close()
}

@Composable
fun HexagonAvatar(imageUrl: String) {
    AsyncImage(
        model = imageUrl, contentDescription = "アバター",
        modifier = Modifier.size(100.dp).clip(HexagonShape).border(2.dp, Color.Gray, HexagonShape),
        contentScale = ContentScale.Crop
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `GenericShape` | カスタムShape定義 |
| `clip` | 描画クリッピング |
| `border` | カスタム形状の枠線 |
| `background` | カスタム形状の背景 |

- `GenericShape`でPath APIによるカスタムShape
- `clip`/`background`/`border`で適用
- ベジェ曲線で滑らかな形状
- 画像のクリッピングに活用

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Path描画](https://zenn.dev/myougatheaxo/articles/android-compose-compose-path-2026)
- [Clip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clip-2026)
- [Border/Shadow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-border-2026)
