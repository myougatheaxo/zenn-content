---
title: "SwipeToDismiss完全ガイド — SwipeToDismissBox/背景アクション/Undo"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**SwipeToDismiss**（SwipeToDismissBox、背景アクション表示、削除Undo、方向制御）を解説します。

---

## SwipeToDismissBox

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeDismissExample() {
    var items by remember { mutableStateOf(List(20) { "Item $it" }) }
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(items, key = { it }) { item ->
                val dismissState = rememberSwipeToDismissBoxState(
                    confirmValueChange = { value ->
                        if (value == SwipeToDismissBoxValue.EndToStart) {
                            val deletedItem = item
                            items = items - item
                            scope.launch {
                                val result = snackbarHostState.showSnackbar(
                                    message = "${deletedItem}を削除",
                                    actionLabel = "元に戻す"
                                )
                                if (result == SnackbarResult.ActionPerformed) {
                                    items = items + deletedItem
                                }
                            }
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
                                .background(Color.Red)
                                .padding(horizontal = 20.dp),
                            contentAlignment = Alignment.CenterEnd
                        ) {
                            Icon(Icons.Default.Delete, "削除", tint = Color.White)
                        }
                    },
                    enableDismissFromStartToEnd = false
                ) {
                    ListItem(
                        headlineContent = { Text(item) },
                        modifier = Modifier.background(MaterialTheme.colorScheme.surface)
                    )
                }
            }
        }
    }
}
```

---

## 双方向スワイプ

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BidirectionalSwipe(item: String, onDelete: () -> Unit, onArchive: () -> Unit) {
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
            val color = when (direction) {
                SwipeToDismissBoxValue.StartToEnd -> Color(0xFF4CAF50)
                SwipeToDismissBoxValue.EndToStart -> Color.Red
                else -> Color.Transparent
            }
            val icon = when (direction) {
                SwipeToDismissBoxValue.StartToEnd -> Icons.Default.Archive
                SwipeToDismissBoxValue.EndToStart -> Icons.Default.Delete
                else -> Icons.Default.Delete
            }
            val alignment = when (direction) {
                SwipeToDismissBoxValue.StartToEnd -> Alignment.CenterStart
                else -> Alignment.CenterEnd
            }
            Box(
                Modifier.fillMaxSize().background(color).padding(horizontal = 20.dp),
                contentAlignment = alignment
            ) { Icon(icon, null, tint = Color.White) }
        }
    ) {
        ListItem(
            headlineContent = { Text(item) },
            modifier = Modifier.background(MaterialTheme.colorScheme.surface)
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SwipeToDismissBox` | スワイプ削除 |
| `SwipeToDismissBoxState` | スワイプ状態管理 |
| `confirmValueChange` | 削除確認 |
| `dismissDirection` | スワイプ方向判定 |

- `SwipeToDismissBox`でスワイプ削除/アーカイブ
- `backgroundContent`で背景アクション表示
- `confirmValueChange`で確認/Undo実装
- `enableDismissFromStartToEnd`で方向制限

---

8種類のAndroidアプリテンプレート（リスト操作対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ジェスチャー](https://zenn.dev/myougatheaxo/articles/android-compose-compose-gesture-2026)
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
- [Snackbar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-snackbar-2026)
