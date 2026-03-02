---
title: "グラデーション完全ガイド — linearGradient/radialGradient/sweepGradient/背景"
emoji: "🌈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**グラデーション**（linearGradient、radialGradient、sweepGradient、テキストグラデーション）を解説します。

---

## Brush.linearGradient

```kotlin
@Composable
fun GradientExamples() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 水平グラデーション
        Box(
            Modifier
                .fillMaxWidth()
                .height(80.dp)
                .background(
                    Brush.horizontalGradient(listOf(Color(0xFF6200EA), Color(0xFF03DAC5))),
                    RoundedCornerShape(12.dp)
                )
        )

        // 斜めグラデーション
        Box(
            Modifier
                .fillMaxWidth()
                .height(80.dp)
                .background(
                    Brush.linearGradient(
                        colors = listOf(Color(0xFFFF5722), Color(0xFFFFC107), Color(0xFF4CAF50)),
                        start = Offset.Zero,
                        end = Offset(Float.POSITIVE_INFINITY, Float.POSITIVE_INFINITY)
                    ),
                    RoundedCornerShape(12.dp)
                )
        )

        // 放射状グラデーション
        Box(
            Modifier
                .size(150.dp)
                .background(
                    Brush.radialGradient(
                        colors = listOf(Color.White, Color(0xFF2196F3)),
                        radius = 300f
                    ),
                    CircleShape
                )
        )

        // 扇形グラデーション
        Box(
            Modifier
                .size(150.dp)
                .background(
                    Brush.sweepGradient(
                        listOf(Color.Red, Color.Yellow, Color.Green, Color.Cyan, Color.Blue, Color.Red)
                    ),
                    CircleShape
                )
        )
    }
}
```

---

## グラデーションボタン

```kotlin
@Composable
fun GradientButton(text: String, onClick: () -> Unit) {
    val gradient = Brush.horizontalGradient(
        listOf(Color(0xFF6200EA), Color(0xFFBB86FC))
    )

    Box(
        modifier = Modifier
            .fillMaxWidth()
            .clip(RoundedCornerShape(12.dp))
            .background(gradient)
            .clickable(onClick = onClick)
            .padding(vertical = 14.dp),
        contentAlignment = Alignment.Center
    ) {
        Text(text, color = Color.White, fontWeight = FontWeight.Bold, fontSize = 16.sp)
    }
}

// グラデーションテキスト
@Composable
fun GradientText(text: String) {
    Text(
        text = text,
        style = TextStyle(
            fontSize = 32.sp,
            fontWeight = FontWeight.Bold,
            brush = Brush.linearGradient(
                listOf(Color(0xFFFF5722), Color(0xFF9C27B0))
            )
        )
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Brush.linearGradient` | 線形グラデーション |
| `Brush.radialGradient` | 放射状グラデーション |
| `Brush.sweepGradient` | 扇形グラデーション |
| `Brush.horizontalGradient` | 水平グラデーション |

- `Brush`でグラデーション背景/枠線/テキスト
- `linearGradient`で方向指定のグラデーション
- `radialGradient`で中心から外周へ
- テキストにも`brush`パラメータで適用可能

---

8種類のAndroidアプリテンプレート（カスタムUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Border/Shadow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-border-2026)
- [カスタムCanvas](https://zenn.dev/myougatheaxo/articles/android-compose-custom-canvas-2026)
- [カスタムShape](https://zenn.dev/myougatheaxo/articles/android-compose-compose-shape-2026)
