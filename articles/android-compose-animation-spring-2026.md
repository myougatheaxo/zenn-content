---
title: "Springアニメーション完全ガイド — 物理ベース/バウンス/スナップ"
emoji: "🎢"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

**Springアニメーション**（物理ベース、バウンス効果、スナップ、カスタムspring、Compose連携）を解説します。

---

## spring vs tween

```kotlin
// tween: 時間ベース（一定時間で完了）
val alpha by animateFloatAsState(
    targetValue = if (visible) 1f else 0f,
    animationSpec = tween(durationMillis = 300, easing = FastOutSlowInEasing)
)

// spring: 物理ベース（自然な動き）
val scale by animateFloatAsState(
    targetValue = if (pressed) 0.9f else 1f,
    animationSpec = spring(
        dampingRatio = Spring.DampingRatioMediumBouncy,
        stiffness = Spring.StiffnessMedium
    )
)
```

---

## バウンス効果

```kotlin
@Composable
fun BouncyButton(text: String, onClick: () -> Unit) {
    var isPressed by remember { mutableStateOf(false) }

    val scale by animateFloatAsState(
        targetValue = if (isPressed) 0.85f else 1f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioLowBouncy,  // バウンス多め
            stiffness = Spring.StiffnessLow              // ゆっくり
        )
    )

    Button(
        onClick = onClick,
        modifier = Modifier
            .graphicsLayer { scaleX = scale; scaleY = scale }
            .pointerInput(Unit) {
                detectTapGestures(
                    onPress = {
                        isPressed = true
                        tryAwaitRelease()
                        isPressed = false
                    }
                )
            }
    ) { Text(text) }
}
```

---

## Animatable + spring

```kotlin
@Composable
fun SpringyCard() {
    val offsetY = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()

    Card(
        Modifier
            .offset { IntOffset(0, offsetY.value.roundToInt()) }
            .pointerInput(Unit) {
                detectDragGestures(
                    onDrag = { _, dragAmount ->
                        scope.launch { offsetY.snapTo(offsetY.value + dragAmount.y) }
                    },
                    onDragEnd = {
                        scope.launch {
                            offsetY.animateTo(
                                targetValue = 0f,
                                animationSpec = spring(
                                    dampingRatio = 0.3f,  // 強いバウンス
                                    stiffness = 200f
                                )
                            )
                        }
                    }
                )
            }
            .padding(16.dp)
    ) {
        Text("ドラッグ→バウンス", Modifier.padding(24.dp))
    }
}
```

---

## リスト登場アニメーション

```kotlin
@Composable
fun AnimatedListItem(index: Int, content: @Composable () -> Unit) {
    val offsetY = remember { Animatable(100f) }
    val alpha = remember { Animatable(0f) }

    LaunchedEffect(Unit) {
        delay(index * 50L) // 順番に登場
        launch { offsetY.animateTo(0f, spring(dampingRatio = 0.6f, stiffness = 300f)) }
        launch { alpha.animateTo(1f, tween(300)) }
    }

    Box(
        Modifier
            .offset { IntOffset(0, offsetY.value.roundToInt()) }
            .graphicsLayer { this.alpha = alpha.value }
    ) {
        content()
    }
}
```

---

## dampingRatioの比較

```kotlin
// DampingRatioNoBouncy (1.0f): バウンスなし、素早く停止
// DampingRatioLowBouncy (0.75f): 少しバウンス
// DampingRatioMediumBouncy (0.5f): 中程度バウンス
// DampingRatioHighBouncy (0.2f): 大きくバウンス

@Composable
fun DampingComparison() {
    var trigger by remember { mutableStateOf(false) }
    val ratios = listOf(1.0f, 0.75f, 0.5f, 0.2f)

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { trigger = !trigger }) { Text("Toggle") }

        ratios.forEach { ratio ->
            val offset by animateFloatAsState(
                targetValue = if (trigger) 200f else 0f,
                animationSpec = spring(dampingRatio = ratio, stiffness = Spring.StiffnessLow)
            )
            Box(
                Modifier.offset(x = offset.dp).size(40.dp)
                    .background(MaterialTheme.colorScheme.primary, CircleShape)
            )
        }
    }
}
```

---

## まとめ

| パラメータ | 効果 |
|-----------|------|
| dampingRatio低 | バウンス多 |
| dampingRatio高 | バウンス少 |
| stiffness低 | ゆっくり |
| stiffness高 | 素早く |

- `spring()`で物理ベースの自然なアニメーション
- `dampingRatio`でバウンス量を制御
- `stiffness`でアニメーション速度を制御
- ボタンプレス、ドラッグ解放、リスト登場に最適

---

8種類のAndroidアプリテンプレート（アニメーション対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アニメーション基本](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [Material3 Motion](https://zenn.dev/myougatheaxo/articles/android-compose-material3-motion-2026)
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-gesture-touch-2026)
