---
title: "Composeジェスチャー入門 — タップ・スワイプ・ドラッグの実装パターン"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

Composeでタップ、長押し、スワイプ、ドラッグなどの**ジェスチャー操作**を実装する方法を解説します。

---

## タップ検出（clickable）

```kotlin
@Composable
fun TappableCard() {
    var tapped by remember { mutableStateOf(false) }

    Card(
        modifier = Modifier
            .fillMaxWidth()
            .clickable { tapped = !tapped }
            .padding(16.dp),
        colors = CardDefaults.cardColors(
            containerColor = if (tapped)
                MaterialTheme.colorScheme.primaryContainer
            else
                MaterialTheme.colorScheme.surface
        )
    ) {
        Text("タップしてね", Modifier.padding(16.dp))
    }
}
```

---

## 長押し（combinedClickable）

```kotlin
@Composable
fun LongPressItem(onLongPress: () -> Unit) {
    Text(
        text = "長押しで削除",
        modifier = Modifier
            .combinedClickable(
                onClick = { /* 通常タップ */ },
                onLongClick = { onLongPress() },
                onDoubleClick = { /* ダブルタップ */ }
            )
            .padding(16.dp)
    )
}
```

`combinedClickable`で**タップ・長押し・ダブルタップ**を一括管理。

---

## スワイプで削除（SwipeToDismiss）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeableItem(
    item: String,
    onDelete: () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value == SwipeToDismissBoxValue.EndToStart) {
                onDelete()
                true
            } else false
        }
    )

    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            Box(
                Modifier
                    .fillMaxSize()
                    .background(MaterialTheme.colorScheme.error)
                    .padding(16.dp),
                contentAlignment = Alignment.CenterEnd
            ) {
                Icon(Icons.Default.Delete, "削除", tint = Color.White)
            }
        }
    ) {
        Card(Modifier.fillMaxWidth()) {
            Text(item, Modifier.padding(16.dp))
        }
    }
}
```

---

## ドラッグ（draggable）

```kotlin
@Composable
fun DraggableBox() {
    var offsetX by remember { mutableFloatStateOf(0f) }

    Box(
        Modifier
            .offset { IntOffset(offsetX.roundToInt(), 0) }
            .draggable(
                orientation = Orientation.Horizontal,
                state = rememberDraggableState { delta ->
                    offsetX += delta
                }
            )
            .size(80.dp)
            .background(
                MaterialTheme.colorScheme.primary,
                RoundedCornerShape(16.dp)
            )
    )
}
```

水平方向のドラッグ。`Orientation.Vertical`で縦方向も可。

---

## 2D自由ドラッグ（pointerInput）

```kotlin
@Composable
fun FreeDragBox() {
    var offset by remember { mutableStateOf(Offset.Zero) }

    Box(
        Modifier
            .offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
            .pointerInput(Unit) {
                detectDragGestures { change, dragAmount ->
                    change.consume()
                    offset += dragAmount
                }
            }
            .size(80.dp)
            .background(
                MaterialTheme.colorScheme.tertiary,
                CircleShape
            )
    )
}
```

`detectDragGestures`で**X・Y両方向の自由移動**を実装。

---

## ピンチズーム

```kotlin
@Composable
fun ZoomableImage(imageUrl: String) {
    var scale by remember { mutableFloatStateOf(1f) }

    AsyncImage(
        model = imageUrl,
        contentDescription = "ズーム可能な画像",
        modifier = Modifier
            .fillMaxWidth()
            .pointerInput(Unit) {
                detectTransformGestures { _, _, zoom, _ ->
                    scale = (scale * zoom).coerceIn(0.5f, 3f)
                }
            }
            .graphicsLayer(scaleX = scale, scaleY = scale)
    )
}
```

---

## スクロール方向検出

```kotlin
@Composable
fun ScrollDirectionDetector() {
    val listState = rememberLazyListState()
    var isScrollingUp by remember { mutableStateOf(false) }

    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemScrollOffset }
            .collect { offset ->
                // スクロール方向でFABの表示/非表示を制御
            }
    }

    // スクロール方向でFABを表示/非表示
    val showFab by remember {
        derivedStateOf { !listState.isScrollInProgress || isScrollingUp }
    }
}
```

---

## まとめ

| ジェスチャー | 実装方法 |
|------------|---------|
| タップ | `clickable` |
| 長押し・ダブルタップ | `combinedClickable` |
| スワイプ削除 | `SwipeToDismissBox` |
| 1軸ドラッグ | `draggable` |
| 自由ドラッグ | `pointerInput` + `detectDragGestures` |
| ピンチズーム | `pointerInput` + `detectTransformGestures` |

---

8種類のAndroidアプリテンプレート（ジェスチャー対応UI設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Animation入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
