---
title: "ReorderableList完全ガイド — ドラッグ並べ替え/ハプティクス/永続化"
emoji: "↕️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "dragdrop"]
published: true
---

## この記事で学べること

**ReorderableList**（ドラッグ並べ替え、ハプティクスフィードバック、並び順の永続化）を解説します。

---

## ドラッグ並べ替え

```kotlin
@Composable
fun ReorderableListDemo() {
    val items = remember { mutableStateListOf("タスク1", "タスク2", "タスク3", "タスク4", "タスク5") }
    var draggedIndex by remember { mutableIntStateOf(-1) }
    var dragOffset by remember { mutableFloatStateOf(0f) }

    LazyColumn(Modifier.fillMaxSize().padding(16.dp)) {
        itemsIndexed(items, key = { _, item -> item }) { index, item ->
            val haptic = LocalHapticFeedback.current

            Card(
                Modifier
                    .fillMaxWidth()
                    .padding(vertical = 4.dp)
                    .graphicsLayer {
                        if (index == draggedIndex) {
                            translationY = dragOffset
                            scaleX = 1.05f; scaleY = 1.05f
                            shadowElevation = 8f
                        }
                    }
                    .pointerInput(Unit) {
                        detectDragGesturesAfterLongPress(
                            onDragStart = {
                                draggedIndex = index
                                haptic.performHapticFeedback(HapticFeedbackType.LongPress)
                            },
                            onDrag = { _, dragAmount ->
                                dragOffset += dragAmount.y
                                val targetIndex = (index + (dragOffset / 80.dp.toPx()).toInt())
                                    .coerceIn(0, items.lastIndex)
                                if (targetIndex != index && targetIndex != draggedIndex) {
                                    items.apply {
                                        add(targetIndex, removeAt(draggedIndex))
                                    }
                                    draggedIndex = targetIndex
                                    dragOffset = 0f
                                }
                            },
                            onDragEnd = { draggedIndex = -1; dragOffset = 0f },
                            onDragCancel = { draggedIndex = -1; dragOffset = 0f }
                        )
                    }
            ) {
                Row(Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                    Icon(Icons.Default.DragHandle, "並べ替え", tint = Color.Gray)
                    Spacer(Modifier.width(12.dp))
                    Text(item, style = MaterialTheme.typography.bodyLarge)
                }
            }
        }
    }
}
```

---

## チェックリスト並べ替え

```kotlin
data class CheckItem(val id: Int, val text: String, val checked: Boolean = false)

@Composable
fun ReorderableChecklist() {
    val items = remember {
        mutableStateListOf(
            CheckItem(1, "買い物"), CheckItem(2, "掃除"),
            CheckItem(3, "料理"), CheckItem(4, "洗濯")
        )
    }

    LazyColumn(Modifier.padding(16.dp)) {
        itemsIndexed(items, key = { _, item -> item.id }) { index, item ->
            Row(
                Modifier.fillMaxWidth().padding(vertical = 4.dp)
                    .animateItem(),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Icon(Icons.Default.DragHandle, null, tint = Color.Gray)
                Checkbox(
                    checked = item.checked,
                    onCheckedChange = { items[index] = item.copy(checked = it) }
                )
                Text(
                    item.text,
                    textDecoration = if (item.checked) TextDecoration.LineThrough else null,
                    color = if (item.checked) Color.Gray else Color.Unspecified
                )
            }
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| ドラッグ検出 | `detectDragGesturesAfterLongPress` |
| 視覚効果 | `graphicsLayer`で拡大+影 |
| ハプティクス | `HapticFeedbackType.LongPress` |
| アニメーション | `animateItem()` |

- `detectDragGesturesAfterLongPress`で長押し後ドラッグ
- `graphicsLayer`でドラッグ中の視覚フィードバック
- `mutableStateListOf`で並び順を管理
- ハプティクスフィードバックでUX向上

---

8種類のAndroidアプリテンプレート（タスク管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DragDrop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-drag-drop-2026)
- [SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-compose-swipe-dismiss-2026)
- [StickyHeader](https://zenn.dev/myougatheaxo/articles/android-compose-compose-sticky-header-2026)
