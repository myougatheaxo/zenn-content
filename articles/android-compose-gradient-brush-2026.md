---
title: "Composeグラデーション/Brushガイド — 線形/放射/スウィープ"
emoji: "🌈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "design"]
published: true
---

## この記事で学べること

Composeの**Brush/グラデーション**（linearGradient、radialGradient、sweepGradient）を解説します。

---

## 線形グラデーション

```kotlin
@Composable
fun LinearGradientExample() {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .background(
                Brush.linearGradient(
                    colors = listOf(Color(0xFF6200EA), Color(0xFF03DAC5)),
                    start = Offset.Zero,
                    end = Offset(Float.POSITIVE_INFINITY, Float.POSITIVE_INFINITY)
                )
            )
    )
}

// テキストにグラデーション
@Composable
fun GradientText(text: String) {
    Text(
        text = text,
        style = MaterialTheme.typography.headlineLarge.copy(
            brush = Brush.linearGradient(
                colors = listOf(Color(0xFFFF6B6B), Color(0xFF4ECDC4))
            )
        )
    )
}
```

---

## 放射状グラデーション

```kotlin
@Composable
fun RadialGradientExample() {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(
                Brush.radialGradient(
                    colors = listOf(Color.White, Color(0xFF6200EA)),
                    center = Offset(300f, 300f),
                    radius = 400f
                )
            )
    )
}
```

---

## スウィープグラデーション

```kotlin
@Composable
fun SweepGradientExample() {
    Canvas(modifier = Modifier.size(200.dp)) {
        drawCircle(
            brush = Brush.sweepGradient(
                colors = listOf(
                    Color.Red, Color.Yellow, Color.Green,
                    Color.Cyan, Color.Blue, Color.Magenta, Color.Red
                )
            )
        )
    }
}
```

---

## グラデーションボタン

```kotlin
@Composable
fun GradientButton(
    text: String,
    onClick: () -> Unit,
    gradient: Brush = Brush.linearGradient(
        colors = listOf(Color(0xFF6200EA), Color(0xFFBB86FC))
    )
) {
    Button(
        onClick = onClick,
        colors = ButtonDefaults.buttonColors(containerColor = Color.Transparent),
        contentPadding = PaddingValues()
    ) {
        Box(
            modifier = Modifier
                .background(gradient)
                .padding(horizontal = 24.dp, vertical = 12.dp),
            contentAlignment = Alignment.Center
        ) {
            Text(text, color = Color.White)
        }
    }
}
```

---

## グラデーションボーダー

```kotlin
@Composable
fun GradientBorderCard() {
    val gradient = Brush.linearGradient(
        colors = listOf(Color(0xFFFF6B6B), Color(0xFF4ECDC4))
    )

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
            .border(2.dp, gradient, RoundedCornerShape(16.dp)),
        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surface)
    ) {
        Column(Modifier.padding(16.dp)) {
            Text("グラデーションボーダー", style = MaterialTheme.typography.titleMedium)
            Text("カード内のコンテンツ", style = MaterialTheme.typography.bodyMedium)
        }
    }
}
```

---

## まとめ

- `Brush.linearGradient`で線形グラデーション
- `Brush.radialGradient`で放射状グラデーション
- `Brush.sweepGradient`で円形グラデーション
- `TextStyle.brush`でテキストにグラデーション適用
- `Modifier.border`でグラデーションボーダー
- `Canvas`+`drawCircle`でカスタム描画

---

8種類のAndroidアプリテンプレート（モダンUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
- [カスタムレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-custom-layout-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
