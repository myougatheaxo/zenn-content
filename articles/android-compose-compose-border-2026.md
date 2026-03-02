---
title: "Border/Shadow完全ガイド — border/shadow/グラデーション枠/カード影"
emoji: "🖌️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Border/Shadow**（border Modifier、shadow、グラデーション枠線、カードの影）を解説します。

---

## Border

```kotlin
@Composable
fun BorderExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 基本枠線
        Box(
            Modifier
                .size(100.dp)
                .border(2.dp, Color.Black, RoundedCornerShape(8.dp))
                .padding(16.dp)
        ) { Text("基本") }

        // グラデーション枠線
        Box(
            Modifier
                .size(100.dp)
                .border(
                    width = 3.dp,
                    brush = Brush.linearGradient(listOf(Color.Red, Color.Blue)),
                    shape = RoundedCornerShape(16.dp)
                )
                .padding(16.dp)
        ) { Text("グラデ") }

        // 点線風（dashPathEffect）
        Box(
            Modifier
                .size(100.dp)
                .drawBehind {
                    drawRoundRect(
                        color = Color.Gray,
                        style = Stroke(
                            width = 2.dp.toPx(),
                            pathEffect = PathEffect.dashPathEffect(
                                floatArrayOf(10f, 10f), 0f
                            )
                        ),
                        cornerRadius = CornerRadius(8.dp.toPx())
                    )
                }
                .padding(16.dp)
        ) { Text("点線") }
    }
}
```

---

## Shadow

```kotlin
@Composable
fun ShadowExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // shadow Modifier
        Box(
            Modifier
                .fillMaxWidth()
                .shadow(8.dp, RoundedCornerShape(12.dp))
                .background(Color.White)
                .padding(16.dp)
        ) { Text("shadow 8dp") }

        // Card elevation
        Card(
            modifier = Modifier.fillMaxWidth(),
            elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
        ) {
            Text("Card elevation 4dp", Modifier.padding(16.dp))
        }

        // アニメーション影
        var elevated by remember { mutableStateOf(false) }
        val elevation by animateDpAsState(
            if (elevated) 12.dp else 2.dp, label = "elevation"
        )
        Card(
            modifier = Modifier.fillMaxWidth().clickable { elevated = !elevated },
            elevation = CardDefaults.cardElevation(defaultElevation = elevation)
        ) {
            Text("タップで影変化", Modifier.padding(16.dp))
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `border` | 枠線 |
| `shadow` | 影 |
| `Brush` | グラデーション枠 |
| `dashPathEffect` | 点線枠 |

- `border`で枠線の太さ/色/形状を指定
- `Brush`でグラデーション枠線
- `shadow`でドロップシャドウ
- `dashPathEffect`で点線枠を描画

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カスタムShape](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shape-2026)
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
