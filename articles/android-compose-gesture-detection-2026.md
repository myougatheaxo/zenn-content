---
title: "Composeジェスチャー検出ガイド — タップ/ドラッグ/ピンチ/回転"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

Composeの**ジェスチャー検出**（タップ、ロングプレス、ドラッグ、ピンチ、回転）を解説します。

---

## タップ/ロングプレス

```kotlin
@Composable
fun TapDetection() {
    var message by remember { mutableStateOf("") }

    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.LightGray)
            .pointerInput(Unit) {
                detectTapGestures(
                    onTap = { message = "タップ: $it" },
                    onDoubleTap = { message = "ダブルタップ" },
                    onLongPress = { message = "ロングプレス" },
                    onPress = {
                        // プレス開始
                        tryAwaitRelease()
                        // リリース
                    }
                )
            },
        contentAlignment = Alignment.Center
    ) {
        Text(message)
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

    Box(
        Modifier
            .offset { IntOffset(offsetX.roundToInt(), offsetY.roundToInt()) }
            .size(100.dp)
            .background(Color.Blue, RoundedCornerShape(16.dp))
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
```

---

## ピンチズーム

```kotlin
@Composable
fun PinchZoomImage() {
    var scale by remember { mutableFloatStateOf(1f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectTransformGestures { _, pan, zoom, _ ->
                    scale = (scale * zoom).coerceIn(0.5f, 5f)
                    offset += pan
                }
            }
    ) {
        AsyncImage(
            model = "https://example.com/image.jpg",
            contentDescription = null,
            modifier = Modifier
                .fillMaxSize()
                .graphicsLayer(
                    scaleX = scale,
                    scaleY = scale,
                    translationX = offset.x,
                    translationY = offset.y
                )
        )
    }
}
```

---

## スワイプ方向検出

```kotlin
@Composable
fun SwipeDirection() {
    var direction by remember { mutableStateOf("") }

    Box(
        modifier = Modifier
            .fillMaxSize()
            .pointerInput(Unit) {
                detectDragGestures(
                    onDragEnd = { direction = "" }
                ) { change, dragAmount ->
                    change.consume()
                    direction = when {
                        dragAmount.x > 10 -> "→ 右"
                        dragAmount.x < -10 -> "← 左"
                        dragAmount.y > 10 -> "↓ 下"
                        dragAmount.y < -10 -> "↑ 上"
                        else -> direction
                    }
                }
            },
        contentAlignment = Alignment.Center
    ) {
        Text(direction.ifEmpty { "スワイプしてください" })
    }
}
```

---

## まとめ

- `detectTapGestures`でタップ/ダブルタップ/ロングプレス
- `detectDragGestures`でドラッグ
- `detectTransformGestures`でピンチ/ズーム/回転
- `pointerInput(Unit)`でジェスチャーハンドラ登録
- `change.consume()`でイベント消費
- `graphicsLayer`でスケール/移動の反映

---

8種類のAndroidアプリテンプレート（ジェスチャー操作実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ドラッグ並べ替え](https://zenn.dev/myougatheaxo/articles/android-compose-drag-reorder-2026)
- [SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-swipe-dismiss-2026)
- [Canvas描画](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-drawing-2026)
