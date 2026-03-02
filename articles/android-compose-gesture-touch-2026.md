---
title: "ジェスチャー/タッチ操作完全ガイド — pointerInput/swipe/pinch"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**ジェスチャー操作**（タップ、スワイプ、ドラッグ、ピンチズーム、長押し）を解説します。

---

## 基本タップ

```kotlin
Box(Modifier.size(100.dp).clickable { /* タップ */ })

Box(Modifier.size(100.dp).combinedClickable(
    onClick = { /* タップ */ },
    onLongClick = { /* 長押し */ },
    onDoubleClick = { /* ダブルタップ */ }
))
```

---

## ドラッグ

```kotlin
@Composable
fun DraggableBox() {
    var offsetX by remember { mutableFloatStateOf(0f) }
    var offsetY by remember { mutableFloatStateOf(0f) }

    Box(
        modifier = Modifier
            .offset { IntOffset(offsetX.roundToInt(), offsetY.roundToInt()) }
            .size(80.dp)
            .background(MaterialTheme.colorScheme.primary, RoundedCornerShape(8.dp))
            .pointerInput(Unit) {
                detectDragGestures { change, dragAmount ->
                    change.consume()
                    offsetX += dragAmount.x
                    offsetY += dragAmount.y
                }
            },
        contentAlignment = Alignment.Center
    ) { Text("ドラッグ", color = Color.White) }
}
```

---

## ピンチズーム

```kotlin
@Composable
fun ZoomableImage(imageUrl: String) {
    var scale by remember { mutableFloatStateOf(1f) }
    var offset by remember { mutableStateOf(Offset.Zero) }

    val state = rememberTransformableState { zoomChange, panChange, _ ->
        scale = (scale * zoomChange).coerceIn(0.5f, 5f)
        offset += panChange
    }

    AsyncImage(
        model = imageUrl, contentDescription = null,
        modifier = Modifier.fillMaxSize().transformable(state).graphicsLayer {
            scaleX = scale; scaleY = scale
            translationX = offset.x; translationY = offset.y
        },
        contentScale = ContentScale.Fit
    )
}
```

---

## スワイプ削除

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeToDismissItem(item: Item, onDismiss: () -> Unit) {
    val state = rememberSwipeToDismissBoxState(
        confirmValueChange = { if (it == SwipeToDismissBoxValue.EndToStart) { onDismiss(); true } else false }
    )
    SwipeToDismissBox(state = state, backgroundContent = {
        Box(Modifier.fillMaxSize().background(Color.Red).padding(horizontal = 20.dp), contentAlignment = Alignment.CenterEnd) {
            Icon(Icons.Default.Delete, "削除", tint = Color.White)
        }
    }) {
        Card(Modifier.fillMaxWidth()) { Text(item.title, Modifier.padding(16.dp)) }
    }
}
```

---

## まとめ

| ジェスチャー | API |
|-------------|-----|
| タップ | `clickable`, `detectTapGestures` |
| ドラッグ | `detectDragGestures` |
| スワイプ | `SwipeToDismissBox` |
| ピンチ | `transformable` |

- `pointerInput`でカスタムジェスチャー
- `graphicsLayer`で変換適用
- `transformable`でピンチ/パン/回転

---

8種類のAndroidアプリテンプレート（ジェスチャー対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ドラッグ&ドロップ](https://zenn.dev/myougatheaxo/articles/android-compose-drag-drop-reorder-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-canvas-custom-draw-2026)
