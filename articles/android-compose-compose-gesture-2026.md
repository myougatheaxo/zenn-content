---
title: "ジェスチャー完全ガイド — タップ/ドラッグ/スワイプ/マルチタッチ/変換"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**ジェスチャー**（タップ、ドラッグ、スワイプ、マルチタッチ、transformable）を解説します。

---

## タップ検知

```kotlin
@Composable
fun TapDetection() {
    var lastAction by remember { mutableStateOf("なし") }

    Box(
        Modifier
            .size(200.dp)
            .background(MaterialTheme.colorScheme.primaryContainer)
            .pointerInput(Unit) {
                detectTapGestures(
                    onTap = { lastAction = "タップ: $it" },
                    onDoubleTap = { lastAction = "ダブルタップ" },
                    onLongPress = { lastAction = "長押し" },
                    onPress = { /* プレス開始 */ }
                )
            },
        contentAlignment = Alignment.Center
    ) {
        Text(lastAction)
    }
}
```

---

## ドラッグ

```kotlin
@Composable
fun DraggableBox() {
    var offsetX by remember { mutableFloatStateOf(0f) }
    var offsetY by remember { mutableFloatStateOf(0f) }

    Box(Modifier.fillMaxSize()) {
        Box(
            Modifier
                .offset { IntOffset(offsetX.roundToInt(), offsetY.roundToInt()) }
                .size(80.dp)
                .background(Color(0xFF6200EE), RoundedCornerShape(16.dp))
                .pointerInput(Unit) {
                    detectDragGestures { change, dragAmount ->
                        change.consume()
                        offsetX += dragAmount.x
                        offsetY += dragAmount.y
                    }
                },
            contentAlignment = Alignment.Center
        ) {
            Text("ドラッグ", color = Color.White)
        }
    }
}
```

---

## マルチタッチ変換

```kotlin
@Composable
fun TransformableImage() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    val state = rememberTransformableState { zoomChange, panChange, rotationChange ->
        scale = (scale * zoomChange).coerceIn(0.5f, 5f)
        rotation += rotationChange
        offset += panChange
    }

    Image(
        painter = painterResource(R.drawable.sample),
        contentDescription = null,
        modifier = Modifier
            .fillMaxSize()
            .transformable(state = state)
            .graphicsLayer(
                scaleX = scale,
                scaleY = scale,
                rotationZ = rotation,
                translationX = offset.x,
                translationY = offset.y
            )
    )
}
```

---

## スワイプ検知

```kotlin
@Composable
fun SwipeDetection() {
    var direction by remember { mutableStateOf("スワイプしてください") }

    Box(
        Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectHorizontalDragGestures { _, dragAmount ->
                    direction = if (dragAmount > 0) "→ 右スワイプ" else "← 左スワイプ"
                }
            },
        contentAlignment = Alignment.Center
    ) {
        Text(direction, style = MaterialTheme.typography.headlineMedium)
    }
}
```

---

## まとめ

| ジェスチャー | API |
|------------|-----|
| タップ | `detectTapGestures` |
| ドラッグ | `detectDragGestures` |
| マルチタッチ | `transformable` |
| スワイプ | `detectHorizontalDragGestures` |

- `pointerInput`でカスタムジェスチャー検知
- `detectTapGestures`でタップ/ダブルタップ/長押し
- `transformable`でピンチズーム/回転/パン
- `graphicsLayer`で変換を効率的に適用

---

8種類のAndroidアプリテンプレート（ジェスチャー対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ピンチズーム](https://zenn.dev/myougatheaxo/articles/android-compose-pinch-zoom-2026)
- [ドラッグ&ドロップ](https://zenn.dev/myougatheaxo/articles/android-compose-drag-drop-reorder-2026)
- [Swipe Actions](https://zenn.dev/myougatheaxo/articles/android-compose-swipe-action-2026)
