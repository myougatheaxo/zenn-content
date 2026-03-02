---
title: "スワイプアクション完全ガイド — SwipeToDismiss/背景アクション/Undo"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

**スワイプアクション**（SwipeToDismissBox、背景アクション表示、Undo機能、カスタムスワイプ）を解説します。

---

## SwipeToDismissBox

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeableListItem(
    item: Item,
    onDelete: () -> Unit,
    onArchive: () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            when (value) {
                SwipeToDismissBoxValue.EndToStart -> { onDelete(); true }
                SwipeToDismissBoxValue.StartToEnd -> { onArchive(); true }
                else -> false
            }
        }
    )

    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            val direction = dismissState.dismissDirection

            val color by animateColorAsState(
                when (direction) {
                    SwipeToDismissBoxValue.EndToStart -> Color(0xFFFF5252) // 削除: 赤
                    SwipeToDismissBoxValue.StartToEnd -> Color(0xFF4CAF50) // アーカイブ: 緑
                    else -> Color.Transparent
                },
                label = "bgColor"
            )

            Box(
                Modifier.fillMaxSize().background(color).padding(horizontal = 20.dp),
                contentAlignment = when (direction) {
                    SwipeToDismissBoxValue.EndToStart -> Alignment.CenterEnd
                    else -> Alignment.CenterStart
                }
            ) {
                Icon(
                    when (direction) {
                        SwipeToDismissBoxValue.EndToStart -> Icons.Default.Delete
                        else -> Icons.Default.Archive
                    },
                    null, tint = Color.White
                )
            }
        }
    ) {
        Card(Modifier.fillMaxWidth()) {
            ListItem(
                headlineContent = { Text(item.title) },
                supportingContent = { Text(item.subtitle) }
            )
        }
    }
}
```

---

## Undo付きスワイプ削除

```kotlin
@Composable
fun SwipeDeleteWithUndo(
    items: List<Item>,
    onDelete: (Item) -> Unit,
    onUndo: (Item) -> Unit
) {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(items, key = { it.id }) { item ->
                SwipeableListItem(
                    item = item,
                    onDelete = {
                        onDelete(item)
                        scope.launch {
                            val result = snackbarHostState.showSnackbar(
                                message = "${item.title} を削除しました",
                                actionLabel = "元に戻す",
                                duration = SnackbarDuration.Short
                            )
                            if (result == SnackbarResult.ActionPerformed) {
                                onUndo(item)
                            }
                        }
                    },
                    onArchive = { /* アーカイブ処理 */ }
                )
            }
        }
    }
}
```

---

## カスタムスワイプ（部分表示）

```kotlin
@Composable
fun RevealSwipeItem(
    content: @Composable () -> Unit,
    actions: @Composable () -> Unit
) {
    val offsetX = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()
    val actionWidth = 160.dp
    val actionWidthPx = with(LocalDensity.current) { actionWidth.toPx() }

    Box(Modifier.fillMaxWidth().clipToBounds()) {
        // 背景アクション
        Row(
            Modifier.fillMaxHeight().align(Alignment.CenterEnd).width(actionWidth)
        ) {
            actions()
        }

        // メインコンテンツ
        Box(
            Modifier
                .offset { IntOffset(offsetX.value.toInt(), 0) }
                .pointerInput(Unit) {
                    detectHorizontalDragGestures(
                        onDragEnd = {
                            scope.launch {
                                if (offsetX.value < -actionWidthPx / 2) {
                                    offsetX.animateTo(-actionWidthPx, spring())
                                } else {
                                    offsetX.animateTo(0f, spring())
                                }
                            }
                        },
                        onHorizontalDrag = { _, dragAmount ->
                            scope.launch {
                                val newOffset = (offsetX.value + dragAmount).coerceIn(-actionWidthPx, 0f)
                                offsetX.snapTo(newOffset)
                            }
                        }
                    )
                }
        ) {
            content()
        }
    }
}
```

---

## まとめ

| パターン | 用途 |
|----------|------|
| `SwipeToDismissBox` | 削除/アーカイブ |
| Undo Snackbar | 誤操作の取消 |
| Reveal Actions | 部分表示アクション |
| カスタムドラッグ | 自由な実装 |

- `SwipeToDismissBox`で標準的なスワイプ削除
- Snackbarで「元に戻す」機能を提供
- カスタム実装で部分的にアクションを表示
- `spring()`でスナップバックアニメーション

---

8種類のAndroidアプリテンプレート（スワイプUI実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-gesture-touch-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-optimization-2026)
- [Springアニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-spring-2026)
