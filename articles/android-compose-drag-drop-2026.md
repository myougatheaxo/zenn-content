---
title: "ドラッグ&ドロップ実装ガイド — Composeでリスト並び替え"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

Composeで**ドラッグ&ドロップによるリスト並び替え**を実装する方法を解説します。

---

## 基本のドラッグ並び替え

```kotlin
@Composable
fun ReorderableList() {
    var items by remember {
        mutableStateOf(listOf("タスクA", "タスクB", "タスクC", "タスクD", "タスクE"))
    }
    var draggedIndex by remember { mutableIntStateOf(-1) }
    var dragOffset by remember { mutableStateOf(Offset.Zero) }

    LazyColumn(Modifier.fillMaxSize().padding(16.dp)) {
        itemsIndexed(items, key = { _, item -> item }) { index, item ->
            val isDragged = index == draggedIndex

            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(vertical = 4.dp)
                    .graphicsLayer {
                        if (isDragged) {
                            translationY = dragOffset.y
                            scaleX = 1.05f
                            scaleY = 1.05f
                            alpha = 0.9f
                        }
                    }
                    .pointerInput(Unit) {
                        detectDragGesturesAfterLongPress(
                            onDragStart = { draggedIndex = index },
                            onDrag = { change, dragAmount ->
                                change.consume()
                                dragOffset = dragOffset.copy(
                                    y = dragOffset.y + dragAmount.y
                                )
                            },
                            onDragEnd = {
                                draggedIndex = -1
                                dragOffset = Offset.Zero
                            },
                            onDragCancel = {
                                draggedIndex = -1
                                dragOffset = Offset.Zero
                            }
                        )
                    },
                colors = CardDefaults.cardColors(
                    containerColor = if (isDragged)
                        MaterialTheme.colorScheme.primaryContainer
                    else MaterialTheme.colorScheme.surface
                )
            ) {
                Row(
                    Modifier.padding(16.dp),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Icon(Icons.Default.DragHandle, "ドラッグ")
                    Spacer(Modifier.width(12.dp))
                    Text(item, style = MaterialTheme.typography.bodyLarge)
                }
            }
        }
    }
}
```

---

## ドラッグハンドル付きアイテム

```kotlin
@Composable
fun DragHandleItem(
    text: String,
    onDragStart: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(modifier = modifier.fillMaxWidth()) {
        Row(
            Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Icon(
                Icons.Default.DragHandle,
                "並び替え",
                modifier = Modifier
                    .pointerInput(Unit) {
                        detectDragGestures { _, _ ->
                            onDragStart()
                        }
                    },
                tint = MaterialTheme.colorScheme.outline
            )
            Spacer(Modifier.width(12.dp))
            Text(text, modifier = Modifier.weight(1f))
            Checkbox(checked = false, onCheckedChange = {})
        }
    }
}
```

---

## swipeToDismiss（スワイプ削除）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeToDeleteList() {
    var items by remember {
        mutableStateOf(listOf("アイテム1", "アイテム2", "アイテム3", "アイテム4"))
    }

    LazyColumn(Modifier.padding(16.dp)) {
        items(items, key = { it }) { item ->
            val dismissState = rememberSwipeToDismissBoxState(
                confirmValueChange = { value ->
                    if (value == SwipeToDismissBoxValue.EndToStart) {
                        items = items - item
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
                            .background(MaterialTheme.colorScheme.errorContainer)
                            .padding(horizontal = 20.dp),
                        contentAlignment = Alignment.CenterEnd
                    ) {
                        Icon(
                            Icons.Default.Delete,
                            "削除",
                            tint = MaterialTheme.colorScheme.onErrorContainer
                        )
                    }
                },
                enableDismissFromStartToEnd = false
            ) {
                Card(Modifier.fillMaxWidth()) {
                    Text(
                        item,
                        Modifier.padding(16.dp),
                        style = MaterialTheme.typography.bodyLarge
                    )
                }
            }

            Spacer(Modifier.height(4.dp))
        }
    }
}
```

---

## まとめ

- `detectDragGesturesAfterLongPress`で長押しドラッグ
- `graphicsLayer`でドラッグ中の視覚フィードバック
- `SwipeToDismissBox`でスワイプ削除
- ドラッグハンドルアイコンでUX向上
- `key`指定でリスト操作時のパフォーマンス確保

---

8種類のAndroidアプリテンプレート（リスト操作UI設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [ジェスチャー入門](https://zenn.dev/myougatheaxo/articles/compose-gesture-touch-2026)
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
