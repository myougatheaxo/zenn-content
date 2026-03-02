---
title: "Composeアニメーション応用 — AnimatedVisibility/Crossfade/InfiniteTransition"
emoji: "🎭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "animation"]
published: true
---

## この記事で学べること

Composeの**応用アニメーション**（AnimatedVisibility、Crossfade、InfiniteTransition、Animatable）を解説します。

---

## AnimatedVisibility

```kotlin
@Composable
fun AnimatedVisibilityExample() {
    var visible by remember { mutableStateOf(true) }

    Column(Modifier.padding(16.dp)) {
        Button(onClick = { visible = !visible }) {
            Text(if (visible) "非表示" else "表示")
        }

        AnimatedVisibility(
            visible = visible,
            enter = slideInVertically() + fadeIn(),
            exit = slideOutVertically() + fadeOut()
        ) {
            Card(Modifier.fillMaxWidth().padding(top = 16.dp)) {
                Text("アニメーション付きコンテンツ", Modifier.padding(16.dp))
            }
        }
    }
}

// カスタムエンター/エグジット
AnimatedVisibility(
    visible = visible,
    enter = expandVertically(
        expandFrom = Alignment.Top,
        animationSpec = tween(500)
    ) + fadeIn(animationSpec = tween(500)),
    exit = shrinkVertically(
        shrinkTowards = Alignment.Top,
        animationSpec = tween(300)
    ) + fadeOut(animationSpec = tween(300))
) {
    content()
}
```

---

## Crossfade

```kotlin
@Composable
fun CrossfadeExample() {
    var currentTab by remember { mutableStateOf(0) }

    Column {
        TabRow(selectedTabIndex = currentTab) {
            Tab(selected = currentTab == 0, onClick = { currentTab = 0 }) { Text("ホーム", Modifier.padding(16.dp)) }
            Tab(selected = currentTab == 1, onClick = { currentTab = 1 }) { Text("設定", Modifier.padding(16.dp)) }
        }

        Crossfade(
            targetState = currentTab,
            animationSpec = tween(300),
            label = "tab_crossfade"
        ) { tab ->
            when (tab) {
                0 -> HomeContent()
                1 -> SettingsContent()
            }
        }
    }
}
```

---

## AnimatedContent

```kotlin
@Composable
fun AnimatedCounter() {
    var count by remember { mutableIntStateOf(0) }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        AnimatedContent(
            targetState = count,
            transitionSpec = {
                if (targetState > initialState) {
                    slideInVertically { -it } + fadeIn() togetherWith
                    slideOutVertically { it } + fadeOut()
                } else {
                    slideInVertically { it } + fadeIn() togetherWith
                    slideOutVertically { -it } + fadeOut()
                }
            },
            label = "counter"
        ) { target ->
            Text(
                "$target",
                style = MaterialTheme.typography.displayLarge
            )
        }

        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            Button(onClick = { count-- }) { Text("-") }
            Button(onClick = { count++ }) { Text("+") }
        }
    }
}
```

---

## InfiniteTransition

```kotlin
@Composable
fun PulsingIndicator() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulse")

    val scale by infiniteTransition.animateFloat(
        initialValue = 0.8f,
        targetValue = 1.2f,
        animationSpec = infiniteRepeatable(
            animation = tween(800, easing = EaseInOutCubic),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )

    val alpha by infiniteTransition.animateFloat(
        initialValue = 0.5f,
        targetValue = 1f,
        animationSpec = infiniteRepeatable(
            animation = tween(800),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Box(
        Modifier
            .size(60.dp)
            .scale(scale)
            .alpha(alpha)
            .background(Color.Red, CircleShape),
        contentAlignment = Alignment.Center
    ) {
        Text("●", color = Color.White, fontSize = 24.sp)
    }
}
```

---

## Animatable

```kotlin
@Composable
fun DraggableCard() {
    val offsetX = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()

    Box(
        Modifier
            .offset { IntOffset(offsetX.value.roundToInt(), 0) }
            .pointerInput(Unit) {
                detectHorizontalDragGestures(
                    onDragEnd = {
                        scope.launch {
                            offsetX.animateTo(
                                targetValue = 0f,
                                animationSpec = spring(
                                    dampingRatio = Spring.DampingRatioMediumBouncy,
                                    stiffness = Spring.StiffnessLow
                                )
                            )
                        }
                    }
                ) { _, dragAmount ->
                    scope.launch {
                        offsetX.snapTo(offsetX.value + dragAmount)
                    }
                }
            }
            .size(150.dp)
            .background(MaterialTheme.colorScheme.primary, RoundedCornerShape(16.dp)),
        contentAlignment = Alignment.Center
    ) {
        Text("ドラッグ", color = Color.White)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AnimatedVisibility` | 表示/非表示のアニメーション |
| `Crossfade` | コンテンツ切り替え（フェード） |
| `AnimatedContent` | コンテンツ切り替え（スライド等） |
| `InfiniteTransition` | 無限ループアニメーション |
| `Animatable` | 低レベル制御（ドラッグ・snap） |

---

8種類のAndroidアプリテンプレート（アニメーション設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [SharedElement遷移ガイド](https://zenn.dev/myougatheaxo/articles/android-shared-element-transition-2026)
- [ジェスチャー入門](https://zenn.dev/myougatheaxo/articles/compose-gesture-touch-2026)
