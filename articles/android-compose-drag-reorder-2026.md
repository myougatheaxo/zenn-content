---
title: "ドラッグ並べ替えガイド — LazyColumn reorder実装"
emoji: "↕️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

Composeでの**ドラッグ並べ替え**（LazyColumnリオーダー、長押しドラッグ）を解説します。

---

## 基本の並べ替え

```kotlin
@Composable
fun ReorderableList() {
    val items = remember { mutableStateListOf("項目1", "項目2", "項目3", "項目4", "項目5") }
    var draggedIndex by remember { mutableStateOf<Int?>(null) }

    LazyColumn(Modifier.fillMaxSize()) {
        itemsIndexed(items, key = { _, item -> item }) { index, item ->
            ReorderableItem(
                item = item,
                isDragged = draggedIndex == index,
                onMoveUp = {
                    if (index > 0) {
                        items.add(index - 1, items.removeAt(index))
                    }
                },
                onMoveDown = {
                    if (index < items.lastIndex) {
                        items.add(index + 1, items.removeAt(index))
                    }
                }
            )
        }
    }
}

@Composable
fun ReorderableItem(
    item: String,
    isDragged: Boolean,
    onMoveUp: () -> Unit,
    onMoveDown: () -> Unit
) {
    Card(
        Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 4.dp),
        elevation = CardDefaults.cardElevation(
            defaultElevation = if (isDragged) 8.dp else 1.dp
        )
    ) {
        Row(
            Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Icon(Icons.Default.DragHandle, "並べ替え")
            Spacer(Modifier.width(16.dp))
            Text(item, Modifier.weight(1f))
            Column {
                IconButton(onClick = onMoveUp) {
                    Icon(Icons.Default.KeyboardArrowUp, "上へ")
                }
                IconButton(onClick = onMoveDown) {
                    Icon(Icons.Default.KeyboardArrowDown, "下へ")
                }
            }
        }
    }
}
```

---

## 長押しドラッグ

```kotlin
@Composable
fun DragToReorderList() {
    val items = remember { mutableStateListOf("Apple", "Banana", "Cherry", "Date", "Elderberry") }
    var draggedItem by remember { mutableStateOf<String?>(null) }
    var dragOffset by remember { mutableFloatStateOf(0f) }

    LazyColumn(Modifier.fillMaxSize()) {
        itemsIndexed(items, key = { _, item -> item }) { index, item ->
            val elevation by animateDpAsState(
                if (draggedItem == item) 8.dp else 0.dp,
                label = "elevation"
            )

            Card(
                Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 4.dp)
                    .zIndex(if (draggedItem == item) 1f else 0f)
                    .offset { IntOffset(0, if (draggedItem == item) dragOffset.roundToInt() else 0) }
                    .pointerInput(item) {
                        detectDragGesturesAfterLongPress(
                            onDragStart = { draggedItem = item },
                            onDragEnd = { draggedItem = null; dragOffset = 0f },
                            onDragCancel = { draggedItem = null; dragOffset = 0f },
                            onDrag = { change, dragAmount ->
                                change.consume()
                                dragOffset += dragAmount.y
                                // 簡易的な位置入れ替え
                                val targetIndex = (index + (dragOffset / 80f).roundToInt())
                                    .coerceIn(0, items.lastIndex)
                                if (targetIndex != index) {
                                    items.add(targetIndex, items.removeAt(index))
                                    dragOffset = 0f
                                }
                            }
                        )
                    },
                elevation = CardDefaults.cardElevation(defaultElevation = elevation)
            ) {
                Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.DragHandle, "ドラッグ")
                    Spacer(Modifier.width(16.dp))
                    Text(item, style = MaterialTheme.typography.bodyLarge)
                }
            }
        }
    }
}
```

---

## まとめ

- ボタン式: `onMoveUp`/`onMoveDown`でシンプルな並べ替え
- `detectDragGesturesAfterLongPress`で長押しドラッグ
- `mutableStateListOf`の`add`/`removeAt`で順序変更
- `animateDpAsState`でドラッグ中の影アニメーション
- `zIndex`でドラッグ中のアイテムを最前面に
- `DragHandle`アイコンでドラッグ可能を示す

---

8種類のAndroidアプリテンプレート（リスト操作設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ドラッグ&ドロップガイド](https://zenn.dev/myougatheaxo/articles/android-compose-drag-drop-2026)
- [ジェスチャーガイド](https://zenn.dev/myougatheaxo/articles/compose-gesture-touch-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
