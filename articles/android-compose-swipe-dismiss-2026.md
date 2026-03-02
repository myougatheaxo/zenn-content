---
title: "スワイプ削除/アーカイブガイド — SwipeToDismissBox実装"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "gesture"]
published: true
---

## この記事で学べること

Material3の**SwipeToDismissBox**を使ったスワイプ削除/アーカイブの実装を解説します。

---

## 基本のSwipeToDismiss

```kotlin
@Composable
fun SwipeToDismissExample(
    item: TodoItem,
    onDelete: () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value == SwipeToDismissBoxValue.EndToStart) {
                onDelete()
                true
            } else {
                false
            }
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
        }
    ) {
        ListItem(
            headlineContent = { Text(item.title) },
            supportingContent = { Text(item.description) },
            modifier = Modifier.background(MaterialTheme.colorScheme.surface)
        )
    }
}
```

---

## 双方向スワイプ

```kotlin
@Composable
fun BidirectionalSwipe(
    item: MailItem,
    onArchive: () -> Unit,
    onDelete: () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            when (value) {
                SwipeToDismissBoxValue.StartToEnd -> {
                    onArchive()
                    true
                }
                SwipeToDismissBoxValue.EndToStart -> {
                    onDelete()
                    true
                }
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
                    SwipeToDismissBoxValue.StartToEnd -> Color(0xFF4CAF50)
                    SwipeToDismissBoxValue.EndToStart -> Color(0xFFF44336)
                    else -> Color.Transparent
                },
                label = "bg"
            )
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
                Modifier
                    .fillMaxSize()
                    .background(color)
                    .padding(horizontal = 20.dp),
                contentAlignment = alignment
            ) {
                Icon(icon, null, tint = Color.White)
            }
        },
        enableDismissFromStartToEnd = true,
        enableDismissFromEndToStart = true
    ) {
        ListItem(
            headlineContent = { Text(item.subject) },
            supportingContent = { Text(item.preview) },
            leadingContent = { Icon(Icons.Default.Mail, null) },
            modifier = Modifier.background(MaterialTheme.colorScheme.surface)
        )
    }
}
```

---

## リストでの使用

```kotlin
@Composable
fun SwipeableList(viewModel: TodoViewModel = viewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()

    LazyColumn {
        items(items, key = { it.id }) { item ->
            SwipeToDismissExample(
                item = item,
                onDelete = { viewModel.deleteItem(item.id) }
            )
            HorizontalDivider()
        }
    }
}
```

---

## Undo対応

```kotlin
@Composable
fun SwipeWithUndo(viewModel: TodoViewModel = viewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        LazyColumn(Modifier.padding(padding)) {
            items(items, key = { it.id }) { item ->
                val dismissState = rememberSwipeToDismissBoxState(
                    confirmValueChange = { value ->
                        if (value == SwipeToDismissBoxValue.EndToStart) {
                            viewModel.deleteItem(item.id)
                            scope.launch {
                                val result = snackbarHostState.showSnackbar(
                                    message = "${item.title}を削除しました",
                                    actionLabel = "元に戻す",
                                    duration = SnackbarDuration.Short
                                )
                                if (result == SnackbarResult.ActionPerformed) {
                                    viewModel.restoreItem(item)
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
                                .background(MaterialTheme.colorScheme.error)
                                .padding(horizontal = 20.dp),
                            contentAlignment = Alignment.CenterEnd
                        ) {
                            Icon(Icons.Default.Delete, "削除", tint = Color.White)
                        }
                    }
                ) {
                    ListItem(
                        headlineContent = { Text(item.title) },
                        modifier = Modifier.background(MaterialTheme.colorScheme.surface)
                    )
                }
            }
        }
    }
}
```

---

## まとめ

- `SwipeToDismissBox`でスワイプ削除/アーカイブ
- `rememberSwipeToDismissBoxState`で状態管理
- `confirmValueChange`で確認処理
- 双方向スワイプで削除+アーカイブ
- `animateColorAsState`で背景色アニメーション
- Snackbar + Undoで安全な削除操作

---

8種類のAndroidアプリテンプレート（スワイプ操作設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スワイプアクションガイド](https://zenn.dev/myougatheaxo/articles/android-compose-swipe-actions-2026)
- [ジェスチャーガイド](https://zenn.dev/myougatheaxo/articles/compose-gesture-touch-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
