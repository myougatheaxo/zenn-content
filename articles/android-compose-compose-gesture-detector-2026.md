---
title: "Compose GestureDetector完全ガイド — タップ/長押し/ダブルタップ/ドラッグ検出"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**Compose GestureDetector**（detectTapGestures、detectDragGestures、複合ジェスチャー、PointerInput）を解説します。

---

## タップジェスチャー

```kotlin
@Composable
fun TapGestureDemo() {
    var message by remember { mutableStateOf("タップしてください") }

    Box(
        Modifier.fillMaxWidth().height(200.dp)
            .background(MaterialTheme.colorScheme.primaryContainer, RoundedCornerShape(16.dp))
            .pointerInput(Unit) {
                detectTapGestures(
                    onTap = { offset -> message = "タップ: (${offset.x.toInt()}, ${offset.y.toInt()})" },
                    onDoubleTap = { message = "ダブルタップ!" },
                    onLongPress = { message = "長押し!" },
                    onPress = { /* 押下中のフィードバック */ }
                )
            },
        contentAlignment = Alignment.Center
    ) {
        Text(message, style = MaterialTheme.typography.titleMedium)
    }
}
```

---

## ドラッグジェスチャー

```kotlin
@Composable
fun DragGestureDemo() {
    var offset by remember { mutableStateOf(Offset.Zero) }

    Box(Modifier.fillMaxSize()) {
        Box(
            Modifier.offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
                .size(80.dp)
                .background(MaterialTheme.colorScheme.primary, CircleShape)
                .pointerInput(Unit) {
                    detectDragGestures { change, dragAmount ->
                        change.consume()
                        offset += dragAmount
                    }
                },
            contentAlignment = Alignment.Center
        ) {
            Text("ドラッグ", color = Color.White)
        }
    }
}

// 水平スワイプ検出
@Composable
fun SwipeDetector(onSwipeLeft: () -> Unit, onSwipeRight: () -> Unit) {
    Box(Modifier.fillMaxWidth().height(100.dp)
        .pointerInput(Unit) {
            detectHorizontalDragGestures { _, dragAmount ->
                if (dragAmount > 50) onSwipeRight()
                if (dragAmount < -50) onSwipeLeft()
            }
        })
}
```

---

## 複合ジェスチャー

```kotlin
@Composable
fun ZoomableImage() {
    var scale by remember { mutableFloatStateOf(1f) }
    var rotation by remember { mutableFloatStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    Box(
        Modifier.fillMaxSize()
            .pointerInput(Unit) {
                detectTransformGestures { _, pan, zoom, rotationDelta ->
                    scale = (scale * zoom).coerceIn(0.5f, 3f)
                    rotation += rotationDelta
                    offset += pan
                }
            }
    ) {
        AsyncImage(
            model = "https://example.com/image.jpg",
            contentDescription = null,
            modifier = Modifier.fillMaxSize()
                .graphicsLayer {
                    scaleX = scale
                    scaleY = scale
                    rotationZ = rotation
                    translationX = offset.x
                    translationY = offset.y
                }
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `detectTapGestures` | タップ/長押し |
| `detectDragGestures` | ドラッグ |
| `detectTransformGestures` | ピンチ/回転 |
| `pointerInput` | ジェスチャー検出 |

- `pointerInput(Unit)`でジェスチャー検出ブロックを設定
- `detectTapGestures`でタップ/ダブルタップ/長押し
- `detectDragGestures`でドラッグ量を取得
- `detectTransformGestures`でピンチズーム+回転

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose PointerInput](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pointer-input-2026)
- [Compose Transformable](https://zenn.dev/myougatheaxo/articles/android-compose-compose-transformable-2026)
- [Compose DragDrop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-drag-drop-2026)
