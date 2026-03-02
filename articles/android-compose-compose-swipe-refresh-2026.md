---
title: "Compose SwipeRefresh完全ガイド — PullToRefresh/カスタムインジケーター/状態管理"
emoji: "🔃"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose SwipeRefresh**（PullToRefresh、カスタムインジケーター、ViewModel連携、状態管理）を解説します。

---

## PullToRefresh (Material3)

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PullToRefreshDemo() {
    var items by remember { mutableStateOf(List(20) { "アイテム ${it + 1}" }) }
    var isRefreshing by remember { mutableStateOf(false) }
    val scope = rememberCoroutineScope()

    val pullToRefreshState = rememberPullToRefreshState()

    Box(
        Modifier.fillMaxSize()
            .pullToRefresh(
                isRefreshing = isRefreshing,
                state = pullToRefreshState,
                onRefresh = {
                    scope.launch {
                        isRefreshing = true
                        delay(2000) // データ取得
                        items = List(20) { "更新 ${it + 1} (${System.currentTimeMillis() % 1000})" }
                        isRefreshing = false
                    }
                }
            )
    ) {
        LazyColumn(Modifier.fillMaxSize(), contentPadding = PaddingValues(16.dp)) {
            items(items) { item ->
                ListItem(headlineContent = { Text(item) })
                HorizontalDivider()
            }
        }

        PullToRefreshDefaults.Indicator(
            state = pullToRefreshState,
            isRefreshing = isRefreshing,
            modifier = Modifier.align(Alignment.TopCenter)
        )
    }
}
```

---

## ViewModel連携

```kotlin
@HiltViewModel
class FeedViewModel @Inject constructor(
    private val repository: FeedRepository
) : ViewModel() {
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing = _isRefreshing.asStateFlow()

    val items = MutableStateFlow<List<FeedItem>>(emptyList())

    init { loadData() }

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            items.value = repository.fetchLatest()
            _isRefreshing.value = false
        }
    }

    private fun loadData() {
        viewModelScope.launch {
            items.value = repository.fetchLatest()
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FeedScreen(viewModel: FeedViewModel = hiltViewModel()) {
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()
    val items by viewModel.items.collectAsStateWithLifecycle()
    val pullState = rememberPullToRefreshState()

    Box(
        Modifier.fillMaxSize()
            .pullToRefresh(isRefreshing = isRefreshing, state = pullState,
                onRefresh = { viewModel.refresh() })
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(items) { item ->
                ListItem(
                    headlineContent = { Text(item.title) },
                    supportingContent = { Text(item.description) }
                )
            }
        }
        PullToRefreshDefaults.Indicator(
            state = pullState, isRefreshing = isRefreshing,
            modifier = Modifier.align(Alignment.TopCenter)
        )
    }
}
```

---

## 空状態対応

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RefreshableEmptyState(viewModel: FeedViewModel = hiltViewModel()) {
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()
    val items by viewModel.items.collectAsStateWithLifecycle()
    val pullState = rememberPullToRefreshState()

    Box(
        Modifier.fillMaxSize()
            .pullToRefresh(isRefreshing = isRefreshing, state = pullState,
                onRefresh = { viewModel.refresh() })
    ) {
        if (items.isEmpty() && !isRefreshing) {
            Column(Modifier.fillMaxSize(), verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.CenterHorizontally) {
                Icon(Icons.Default.Inbox, contentDescription = null, modifier = Modifier.size(64.dp))
                Spacer(Modifier.height(16.dp))
                Text("データがありません")
                Text("下にスワイプして更新", color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
        } else {
            LazyColumn(Modifier.fillMaxSize()) {
                items(items) { ListItem(headlineContent = { Text(it.title) }) }
            }
        }
        PullToRefreshDefaults.Indicator(state = pullState, isRefreshing = isRefreshing,
            modifier = Modifier.align(Alignment.TopCenter))
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `pullToRefresh` | プルトゥリフレッシュ |
| `rememberPullToRefreshState` | 状態管理 |
| `Indicator` | リフレッシュ表示 |
| `isRefreshing` | 更新中フラグ |

- `pullToRefresh` Modifierでプルダウン検出
- `PullToRefreshDefaults.Indicator`で標準インジケーター
- ViewModelと連携して`isRefreshing`を管理
- 空状態でもスワイプ更新対応

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose PullRefresh](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pull-refresh-2026)
- [Compose LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-column-2026)
- [Compose Paging3](https://zenn.dev/myougatheaxo/articles/android-compose-compose-paging3-2026)
