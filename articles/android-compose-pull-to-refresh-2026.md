---
title: "Pull-to-Refresh完全ガイド — PullToRefreshBox実装"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**Pull-to-Refresh**（PullToRefreshBox、カスタムインジケーター、状態管理）を解説します。

---

## 基本実装

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableList(viewModel: ItemViewModel = hiltViewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() }
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(items, key = { it.id }) { item ->
                ListItem(
                    headlineContent = { Text(item.title) },
                    supportingContent = { Text(item.description) }
                )
                HorizontalDivider()
            }
        }
    }
}
```

---

## ViewModel

```kotlin
@HiltViewModel
class ItemViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {

    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing = _isRefreshing.asStateFlow()

    val items = repository.observeItems()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            try {
                repository.refreshFromNetwork()
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
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CustomRefreshList() {
    var isRefreshing by remember { mutableStateOf(false) }
    val state = rememberPullToRefreshState()

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = {
            isRefreshing = true
            // データ更新
        },
        state = state,
        indicator = {
            PullToRefreshDefaults.Indicator(
                modifier = Modifier.align(Alignment.TopCenter),
                isRefreshing = isRefreshing,
                state = state,
                color = MaterialTheme.colorScheme.primary
            )
        }
    ) {
        // コンテンツ
        LazyColumn(Modifier.fillMaxSize()) {
            items(20) { index ->
                ListItem(headlineContent = { Text("Item $index") })
            }
        }
    }
}
```

---

## Paging3連携

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PagingRefreshList(viewModel: PagingViewModel = hiltViewModel()) {
    val items = viewModel.items.collectAsLazyPagingItems()
    val isRefreshing = items.loadState.refresh is LoadState.Loading

    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { items.refresh() }
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(items.itemCount, key = items.itemKey { it.id }) { index ->
                items[index]?.let { item ->
                    ListItem(headlineContent = { Text(item.title) })
                }
            }

            if (items.loadState.append is LoadState.Loading) {
                item { CircularProgressIndicator(Modifier.fillMaxWidth().padding(16.dp)) }
            }
        }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `PullToRefreshBox` | プルリフレッシュコンテナ |
| `rememberPullToRefreshState` | 状態管理 |
| `Indicator` | カスタムインジケーター |

- `PullToRefreshBox`で簡単実装
- `isRefreshing`で状態制御
- Paging3の`refresh()`と連携可能
- カスタムインジケーターで独自デザイン

---

8種類のAndroidアプリテンプレート（UI完備）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Paging3](https://zenn.dev/myougatheaxo/articles/android-compose-paging3-compose-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-tips-2026)
- [状態管理](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-advanced-2026)
