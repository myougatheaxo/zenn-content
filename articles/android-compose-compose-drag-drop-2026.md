---
title: "ドラッグ&ドロップ完全ガイド — detectDragGestures/並べ替え/グリッドDnD"
emoji: "🖐️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**ドラッグ&ドロップ**（detectDragGestures、リスト並べ替え、グリッドDnD）を解説します。

---

## 基本ドラッグ

```kotlin
@Composable
fun DraggableBox() {
    var offset by remember { mutableStateOf(Offset.Zero) }

    Box(
        Modifier
            .offset { IntOffset(offset.x.roundToInt(), offset.y.roundToInt()) }
            .size(80.dp)
            .background(MaterialTheme.colorScheme.primary, RoundedCornerShape(12.dp))
            .pointerInput(Unit) {
                detectDragGestures { change, dragAmount ->
                    change.consume()
                    offset += dragAmount
                }
            },
        contentAlignment = Alignment.Center
    ) {
        Text("Drag", color = Color.White)
    }
}
```

---

## リスト並べ替え

```kotlin
@Composable
fun ReorderableList() {
    var items by remember { mutableStateOf(List(10) { "Item ${it + 1}" }) }
    var draggedIndex by remember { mutableIntStateOf(-1) }
    var dragOffset by remember { mutableStateOf(0f) }

    LazyColumn(Modifier.fillMaxSize().padding(16.dp)) {
        itemsIndexed(items, key = { _, item -> item }) { index, item ->
            val elevation by animateDpAsState(
                if (draggedIndex == index) 8.dp else 1.dp, label = "elevation"
            )

            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(vertical = 4.dp)
                    .graphicsLayer {
                        if (draggedIndex == index) translationY = dragOffset
                    }
                    .pointerInput(Unit) {
                        detectDragGesturesAfterLongPress(
                            onDragStart = { draggedIndex = index },
                            onDrag = { change, dragAmount ->
                                change.consume()
                                dragOffset += dragAmount.y
                                val targetIndex = (index + (dragOffset / 80).roundToInt())
                                    .coerceIn(0, items.lastIndex)
                                if (targetIndex != index) {
                                    items = items.toMutableList().apply {
                                        add(targetIndex, removeAt(index))
                                    }
                                    draggedIndex = targetIndex
                                    dragOffset = 0f
                                }
                            },
                            onDragEnd = { draggedIndex = -1; dragOffset = 0f },
                            onDragCancel = { draggedIndex = -1; dragOffset = 0f }
                        )
                    },
                elevation = CardDefaults.cardElevation(defaultElevation = elevation)
            ) {
                ListItem(
                    headlineContent = { Text(item) },
                    leadingContent = { Icon(Icons.Default.DragHandle, "並べ替え") }
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `detectDragGestures` | ドラッグ検知 |
| `detectDragGesturesAfterLongPress` | 長押し後ドラッグ |
| `graphicsLayer` | ドラッグ中の移動表示 |
| `pointerInput` | タッチ入力処理 |

- `detectDragGestures`で自由ドラッグ
- `detectDragGesturesAfterLongPress`でリスト並べ替え
- `graphicsLayer`でドラッグ中のビジュアルフィードバック
- `change.consume()`で親へのイベント伝播を防止

---

8種類のAndroidアプリテンプレート（ジェスチャー対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-2026)
- [SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-compose-swipe-dismiss-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
