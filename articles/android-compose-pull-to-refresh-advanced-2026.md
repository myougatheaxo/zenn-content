---
title: "Pull-to-Refresh実装ガイド — Material3対応/カスタムインジケーター"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**PullToRefreshBox**を使ったPull-to-Refresh実装を解説します。

---

## 基本のPullToRefresh

```kotlin
@Composable
fun RefreshableList(viewModel: ItemViewModel = viewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() }
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(items, key = { it.id }) { item ->
                ListItem(headlineContent = { Text(item.title) })
            }
        }
    }
}
```

---

## ViewModel

```kotlin
class ItemViewModel(private val repository: ItemRepository) : ViewModel() {

    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()

    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items.asStateFlow()

    init { loadItems() }

    private fun loadItems() {
        viewModelScope.launch {
            _items.value = repository.getItems()
        }
    }

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            try {
                _items.value = repository.getItems()
            } finally {
                _isRefreshing.value = false
            }
        }
    }
}
```

---

## カスタムインジケーター

```kotlin
@Composable
fun CustomRefreshList() {
    var isRefreshing by remember { mutableStateOf(false) }
    val state = rememberPullToRefreshState()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = {
            isRefreshing = true
            // 更新処理
        },
        state = state,
        indicator = {
            PullToRefreshDefaults.Indicator(
                state = state,
                isRefreshing = isRefreshing,
                modifier = Modifier.align(Alignment.TopCenter),
                color = MaterialTheme.colorScheme.tertiary
            )
        }
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            // コンテンツ
        }
    }
}
```

---

## エラーハンドリング付き

```kotlin
@Composable
fun RefreshWithError(viewModel: ItemViewModel = viewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()
    val error by viewModel.error.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(error) {
        error?.let {
            snackbarHostState.showSnackbar(
                message = it,
                actionLabel = "再試行",
                duration = SnackbarDuration.Short
            ).let { result ->
                if (result == SnackbarResult.ActionPerformed) {
                    viewModel.refresh()
                }
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        PullToRefreshBox(
            isRefreshing = isRefreshing,
            onRefresh = { viewModel.refresh() },
            modifier = Modifier.padding(padding)
        ) {
            if (items.isEmpty() && !isRefreshing) {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("データがありません\n下に引いて更新", textAlign = TextAlign.Center)
                }
            } else {
                LazyColumn(Modifier.fillMaxSize()) {
                    items(items, key = { it.id }) { item ->
                        ListItem(headlineContent = { Text(item.title) })
                    }
                }
            }
        }
    }
}
```

---

## まとめ

- `PullToRefreshBox`でMaterial3対応のPull-to-Refresh
- `isRefreshing`と`onRefresh`で状態管理
- `rememberPullToRefreshState`でカスタムインジケーター
- ViewModelで`_isRefreshing`を管理
- エラー時はSnackbarで再試行を提供
- 空状態との組み合わせ表示

---

8種類のAndroidアプリテンプレート（Pull-to-Refresh実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn最適化ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
- [エラーハンドリングUIガイド](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
- [Scaffold実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-scaffold-2026)
