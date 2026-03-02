---
title: "PullToRefresh完全ガイド — pullToRefresh/カスタムインジケーター/状態管理"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**PullToRefresh**（pullToRefresh Modifier、カスタムインジケーター、リフレッシュ状態管理）を解説します。

---

## 基本PullToRefresh

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PullRefreshExample(viewModel: ItemViewModel = hiltViewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()
    val pullToRefreshState = rememberPullToRefreshState()

    Box(
        Modifier
            .fillMaxSize()
            .pullToRefresh(
                isRefreshing = isRefreshing,
                state = pullToRefreshState,
                onRefresh = viewModel::refresh
            )
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(items) { item ->
                ListItem(headlineContent = { Text(item.title) })
            }
        }

        PullToRefreshDefaults.Indicator(
            state = pullToRefreshState,
            isRefreshing = isRefreshing,
            modifier = Modifier.align(Alignment.TopCenter)
        )
    }
}

@HiltViewModel
class ItemViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing = _isRefreshing.asStateFlow()

    val items = repository.getItems()
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            repository.refreshFromNetwork()
            _isRefreshing.value = false
        }
    }
}
```

---

## カスタムインジケーター

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CustomIndicatorRefresh() {
    var isRefreshing by remember { mutableStateOf(false) }
    val state = rememberPullToRefreshState()
    val scope = rememberCoroutineScope()

    Box(
        Modifier
            .fillMaxSize()
            .pullToRefresh(
                isRefreshing = isRefreshing,
                state = state,
                onRefresh = {
                    scope.launch {
                        isRefreshing = true
                        delay(2000) // API呼び出し
                        isRefreshing = false
                    }
                }
            )
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(30) { Text("Item $it", Modifier.padding(16.dp)) }
        }

        // カスタムインジケーター
        val scaleFraction = {
            if (isRefreshing) 1f
            else LinearOutSlowInEasing.transform(state.distanceFraction.coerceIn(0f, 1f))
        }

        Box(
            Modifier
                .align(Alignment.TopCenter)
                .padding(top = 16.dp)
                .graphicsLayer {
                    scaleX = scaleFraction()
                    scaleY = scaleFraction()
                }
        ) {
            if (isRefreshing) {
                CircularProgressIndicator(Modifier.size(36.dp))
            } else {
                Icon(
                    Icons.Default.ArrowDownward,
                    contentDescription = null,
                    modifier = Modifier.size(36.dp)
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `pullToRefresh` | リフレッシュModifier |
| `PullToRefreshState` | 引っ張り状態管理 |
| `PullToRefreshDefaults.Indicator` | Material3インジケーター |
| `distanceFraction` | 引っ張り量取得 |

- `pullToRefresh` ModifierでPull-to-Refresh実装
- `PullToRefreshDefaults.Indicator`で標準デザイン
- `distanceFraction`でカスタムアニメーション
- ViewModelで`isRefreshing`状態管理

---

8種類のAndroidアプリテンプレート（リフレッシュ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-2026)
- [Paging3](https://zenn.dev/myougatheaxo/articles/android-compose-paging3-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
