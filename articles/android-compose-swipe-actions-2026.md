---
title: "スワイプアクション実装ガイド — Composeでスワイプ操作UI"
emoji: "👆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

Composeで**スワイプで表示されるアクションボタン**（メール風のアーカイブ・削除UI）を実装する方法を解説します。

---

## SwipeToDismissBox（基本）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeableItem(
    item: Item,
    onDelete: () -> Unit,
    onArchive: () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            when (value) {
                SwipeToDismissBoxValue.EndToStart -> {
                    onDelete()
                    true
                }
                SwipeToDismissBoxValue.StartToEnd -> {
                    onArchive()
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
                when (dismissState.targetValue) {
                    SwipeToDismissBoxValue.EndToStart ->
                        MaterialTheme.colorScheme.errorContainer
                    SwipeToDismissBoxValue.StartToEnd ->
                        Color(0xFF4CAF50)
                    else -> Color.Transparent
                }
            )

            Box(
                Modifier
                    .fillMaxSize()
                    .background(color)
                    .padding(horizontal = 20.dp)
            ) {
                if (direction == SwipeToDismissBoxValue.StartToEnd) {
                    Icon(
                        Icons.Default.Archive,
                        "アーカイブ",
                        Modifier.align(Alignment.CenterStart),
                        tint = Color.White
                    )
                }
                if (direction == SwipeToDismissBoxValue.EndToStart) {
                    Icon(
                        Icons.Default.Delete,
                        "削除",
                        Modifier.align(Alignment.CenterEnd),
                        tint = MaterialTheme.colorScheme.onErrorContainer
                    )
                }
            }
        }
    ) {
        Card(Modifier.fillMaxWidth()) {
            ListItem(
                headlineContent = { Text(item.title) },
                supportingContent = { Text(item.description) }
            )
        }
    }
}
```

---

## スワイプ可能リスト

```kotlin
@Composable
fun SwipeableList(viewModel: ItemViewModel) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        LazyColumn(contentPadding = padding) {
            items(items, key = { it.id }) { item ->
                SwipeableItem(
                    item = item,
                    onDelete = {
                        viewModel.delete(item)
                        scope.launch {
                            val result = snackbarHostState.showSnackbar(
                                "削除しました",
                                actionLabel = "元に戻す"
                            )
                            if (result == SnackbarResult.ActionPerformed) {
                                viewModel.restore(item)
                            }
                        }
                    },
                    onArchive = {
                        viewModel.archive(item)
                    }
                )
            }
        }
    }
}
```

---

## 片方向のみスワイプ

```kotlin
SwipeToDismissBox(
    state = dismissState,
    backgroundContent = { /* ... */ },
    enableDismissFromStartToEnd = false,  // 右スワイプ無効
    enableDismissFromEndToStart = true    // 左スワイプのみ有効
) {
    // コンテンツ
}
```

---

## まとめ

- `SwipeToDismissBox`でスワイプアクション
- `dismissDirection`で方向に応じた背景表示
- `animateColorAsState`でスワイプ中の色変化
- `enableDismissFromStartToEnd/EndToStart`で方向制限
- Snackbar + `元に戻す`でUndoパターン

---

8種類のAndroidアプリテンプレート（スワイプアクション対応設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ドラッグ&ドロップ実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-drag-drop-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Snackbar完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-snackbar-2026)
