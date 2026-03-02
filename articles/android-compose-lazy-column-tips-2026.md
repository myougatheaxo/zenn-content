---
title: "LazyColumn実践Tips — ヘッダー/スティッキー/スクロール制御"
emoji: "📜"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "lazycolumn"]
published: true
---

## この記事で学べること

**LazyColumn**の実践テクニック（stickyHeader、スクロール制御、アイテムアニメーション）を解説します。

---

## stickyHeader

```kotlin
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun GroupedList(groups: Map<String, List<Item>>) {
    LazyColumn {
        groups.forEach { (header, items) ->
            stickyHeader {
                Text(
                    header,
                    modifier = Modifier
                        .fillMaxWidth()
                        .background(MaterialTheme.colorScheme.surfaceVariant)
                        .padding(horizontal = 16.dp, vertical = 8.dp),
                    style = MaterialTheme.typography.titleSmall,
                    fontWeight = FontWeight.Bold
                )
            }
            items(items, key = { it.id }) { item ->
                ListItem(headlineContent = { Text(item.name) })
            }
        }
    }
}
```

---

## スクロール制御

```kotlin
@Composable
fun ScrollControlList(items: List<Item>) {
    val listState = rememberLazyListState()
    val scope = rememberCoroutineScope()

    // スクロール位置の監視
    val showScrollToTop by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 5 }
    }

    Box(Modifier.fillMaxSize()) {
        LazyColumn(state = listState) {
            items(items, key = { it.id }) { item ->
                ListItem(headlineContent = { Text(item.name) })
            }
        }

        // トップへ戻るFAB
        AnimatedVisibility(
            visible = showScrollToTop,
            modifier = Modifier
                .align(Alignment.BottomEnd)
                .padding(16.dp)
        ) {
            FloatingActionButton(onClick = {
                scope.launch { listState.animateScrollToItem(0) }
            }) {
                Icon(Icons.Default.KeyboardArrowUp, "トップへ")
            }
        }
    }
}
```

---

## アイテムアニメーション

```kotlin
@Composable
fun AnimatedItemList(items: List<Item>) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            ListItem(
                headlineContent = { Text(item.name) },
                modifier = Modifier.animateItem(
                    fadeInSpec = tween(300),
                    fadeOutSpec = tween(300),
                    placementSpec = spring()
                )
            )
        }
    }
}
```

---

## 無限スクロール

```kotlin
@Composable
fun InfiniteScrollList(
    items: List<Item>,
    isLoading: Boolean,
    onLoadMore: () -> Unit
) {
    val listState = rememberLazyListState()

    // 末尾付近で追加読み込み
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisibleItem = listState.layoutInfo.visibleItemsInfo.lastOrNull()
                ?: return@derivedStateOf false
            lastVisibleItem.index >= listState.layoutInfo.totalItemsCount - 3
        }
    }

    LaunchedEffect(shouldLoadMore) {
        if (shouldLoadMore && !isLoading) {
            onLoadMore()
        }
    }

    LazyColumn(state = listState) {
        items(items, key = { it.id }) { item ->
            ListItem(headlineContent = { Text(item.name) })
        }

        if (isLoading) {
            item {
                Box(Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
        }
    }
}
```

---

## ヘッダー+フッター

```kotlin
@Composable
fun ListWithHeaderFooter(items: List<Item>) {
    LazyColumn {
        // ヘッダー
        item {
            Text(
                "全${items.size}件",
                modifier = Modifier.padding(16.dp),
                style = MaterialTheme.typography.titleMedium
            )
        }

        // リスト本体
        items(items, key = { it.id }) { item ->
            ListItem(headlineContent = { Text(item.name) })
            HorizontalDivider()
        }

        // フッター
        item {
            Text(
                "リストの末尾です",
                modifier = Modifier.padding(16.dp).fillMaxWidth(),
                textAlign = TextAlign.Center,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

---

## まとめ

- `stickyHeader`でグループ化リストのヘッダー固定
- `rememberLazyListState`でスクロール位置監視
- `animateScrollToItem`でプログラム的スクロール
- `animateItem`でリスト変更時のアニメーション
- `derivedStateOf`で末尾検出→無限スクロール
- `item{}`でヘッダー/フッター追加

---

8種類のAndroidアプリテンプレート（リストUI最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
- [Paging3 + Room](https://zenn.dev/myougatheaxo/articles/android-compose-paging-room-2026)
- [SwipeToDismiss](https://zenn.dev/myougatheaxo/articles/android-compose-swipe-dismiss-2026)
