---
title: "無限スクロール実装ガイド — ComposeでLazyColumnの自動読み込み"
emoji: "♾️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "paging"]
published: true
---

## この記事で学べること

LazyColumnで**末尾到達時に自動でデータを読み込む無限スクロール**を実装する方法を解説します。

---

## 基本の無限スクロール

```kotlin
@Composable
fun InfiniteScrollList(viewModel: ItemViewModel) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val listState = rememberLazyListState()

    // 末尾到達を検出
    val shouldLoadMore = remember {
        derivedStateOf {
            val lastVisibleItem = listState.layoutInfo.visibleItemsInfo.lastOrNull()
                ?: return@derivedStateOf false
            lastVisibleItem.index >= listState.layoutInfo.totalItemsCount - 5
        }
    }

    LaunchedEffect(shouldLoadMore) {
        snapshotFlow { shouldLoadMore.value }
            .distinctUntilChanged()
            .filter { it }
            .collect {
                viewModel.loadMore()
            }
    }

    LazyColumn(state = listState) {
        items(items, key = { it.id }) { item ->
            ItemRow(item)
        }

        if (isLoading) {
            item {
                Box(
                    Modifier.fillMaxWidth().padding(16.dp),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
        }
    }
}
```

---

## ViewModel

```kotlin
class ItemViewModel(private val repository: ItemRepository) : ViewModel() {
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items.asStateFlow()

    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    private var currentPage = 0
    private var hasMore = true

    init {
        loadMore()
    }

    fun loadMore() {
        if (_isLoading.value || !hasMore) return

        viewModelScope.launch {
            _isLoading.value = true
            try {
                val newItems = repository.getItems(page = currentPage, limit = 20)
                _items.update { it + newItems }
                currentPage++
                hasMore = newItems.size == 20
            } catch (e: Exception) {
                // エラー処理
            } finally {
                _isLoading.value = false
            }
        }
    }
}
```

---

## Pull-to-Refresh + 無限スクロール

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableInfiniteList(viewModel: ItemViewModel) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()
    val listState = rememberLazyListState()

    val shouldLoadMore = remember {
        derivedStateOf {
            val lastItem = listState.layoutInfo.visibleItemsInfo.lastOrNull()
                ?: return@derivedStateOf false
            lastItem.index >= listState.layoutInfo.totalItemsCount - 5
        }
    }

    LaunchedEffect(shouldLoadMore) {
        snapshotFlow { shouldLoadMore.value }
            .distinctUntilChanged()
            .filter { it }
            .collect { viewModel.loadMore() }
    }

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() }
    ) {
        LazyColumn(state = listState, modifier = Modifier.fillMaxSize()) {
            items(items, key = { it.id }) { item ->
                ItemRow(item)
            }

            if (isLoading) {
                item {
                    Box(Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                        CircularProgressIndicator(Modifier.size(24.dp))
                    }
                }
            }
        }
    }
}
```

---

## まとめ

- `derivedStateOf` + `snapshotFlow`で末尾到達検出
- 残り5件で事前読み込み開始
- `hasMore`フラグで不要なAPIコールを防止
- `Pull-to-Refresh`との組み合わせ
- より本格的な実装は`Paging 3`ライブラリを推奨

---

8種類のAndroidアプリテンプレート（リスト表示設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Paging 3完全ガイド](https://zenn.dev/myougatheaxo/articles/android-paging3-guide-2026)
- [LazyColumn完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-lazycolumn-guide-2026)
- [Pull-to-Refresh実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-pull-refresh-2026)
