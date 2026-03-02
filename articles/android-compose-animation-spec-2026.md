---
title: "ComposeのAnimationSpec完全ガイド — tween/spring/snap"
emoji: "🎭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Composeの**AnimationSpec**（tween、spring、snap、keyframes、repeatable）を解説します。

---

## tween（時間ベース）

```kotlin
@Composable
fun TweenExample() {
    var expanded by remember { mutableStateOf(false) }

    val size by animateDpAsState(
        targetValue = if (expanded) 200.dp else 100.dp,
        animationSpec = tween(
            durationMillis = 500,
            delayMillis = 100,
            easing = FastOutSlowInEasing
        ),
        label = "size"
    )

    Box(
        Modifier
            .size(size)
            .background(Color.Blue)
            .clickable { expanded = !expanded }
    )
}

// 主要なEasing
// LinearEasing: 等速
// FastOutSlowInEasing: 開始速→終了遅（Material標準）
// LinearOutSlowInEasing: 画面進入用
// FastOutLinearInEasing: 画面退出用
// CubicBezierEasing(0.4f, 0.0f, 0.2f, 1.0f): カスタム
```

---

## spring（物理ベース）

```kotlin
@Composable
fun SpringExample() {
    var toggled by remember { mutableStateOf(false) }

    val offset by animateDpAsState(
        targetValue = if (toggled) 200.dp else 0.dp,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        ),
        label = "offset"
    )

    Box(
        Modifier
            .offset(x = offset)
            .size(80.dp)
            .background(Color.Red, CircleShape)
            .clickable { toggled = !toggled }
    )
}

// dampingRatio（減衰比）
// DampingRatioNoBouncy = 1f   // バウンスなし
// DampingRatioLowBouncy = 0.75f
// DampingRatioMediumBouncy = 0.5f
// DampingRatioHighBouncy = 0.2f

// stiffness（剛性）
// StiffnessHigh = 10000f
// StiffnessMedium = 1500f
// StiffnessMediumLow = 400f
// StiffnessLow = 200f
// StiffnessVeryLow = 50f
```

---

## keyframes

```kotlin
@Composable
fun KeyframesExample() {
    var animate by remember { mutableStateOf(false) }

    val offset by animateDpAsState(
        targetValue = if (animate) 300.dp else 0.dp,
        animationSpec = keyframes {
            durationMillis = 1000
            0.dp at 0 using LinearEasing
            150.dp at 300 using FastOutSlowInEasing
            100.dp at 500 // バウンスバック
            300.dp at 1000
        },
        label = "offset"
    )

    Box(
        Modifier
            .offset(x = offset)
            .size(60.dp)
            .background(Color.Green, CircleShape)
            .clickable { animate = !animate }
    )
}
```

---

## repeatable / infiniteRepeatable

```kotlin
@Composable
fun PulseAnimation() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")

    val scale by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 1.2f,
        animationSpec = infiniteRepeatable(
            animation = tween(600),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )

    val alpha by infiniteTransition.animateFloat(
        initialValue = 1f,
        targetValue = 0.5f,
        animationSpec = infiniteRepeatable(
            animation = tween(600),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Box(
        Modifier
            .graphicsLayer(scaleX = scale, scaleY = scale, alpha = alpha)
            .size(80.dp)
            .background(Color.Red, CircleShape)
    )
}
```

---

## snap（即座に変更）

```kotlin
val color by animateColorAsState(
    targetValue = if (selected) Color.Blue else Color.Gray,
    animationSpec = snap(delayMillis = 0), // アニメーションなし
    label = "color"
)
```

---

## まとめ

| AnimationSpec | 特徴 | 使い所 |
|--------------|------|--------|
| `tween` | 時間指定+Easing | 一般的なアニメーション |
| `spring` | 物理ベース/バウンス | 自然な動き |
| `keyframes` | 時間ごとに値指定 | 複雑な軌道 |
| `repeatable` | 繰り返し（回数指定） | パルス/点滅 |
| `infiniteRepeatable` | 無限繰り返し | ローディング |
| `snap` | 即座に変更 | アニメーション不要時 |

---

8種類のAndroidアプリテンプレート（アニメーション実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-animation-advanced-2026)
- [Shared Element遷移](https://zenn.dev/myougatheaxo/articles/android-compose-shared-element-2026)
- [遷移アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-transition-animation-2026)
