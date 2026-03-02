---
title: "Flow debounce完全ガイド — 検索遅延/入力バッファ/スロットリング"
emoji: "⏳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**Flow debounce**（検索入力遅延、debounce/throttle、distinctUntilChanged）を解説します。

---

## debounce検索

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    private val _query = MutableStateFlow("")
    val query = _query.asStateFlow()

    val searchResults = _query
        .debounce(300)
        .distinctUntilChanged()
        .filter { it.length >= 2 }
        .flatMapLatest { query ->
            repository.search(query)
                .catch { emit(emptyList()) }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onQueryChange(query: String) { _query.value = query }
}

@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    val results by viewModel.searchResults.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = query,
            onValueChange = viewModel::onQueryChange,
            label = { Text("検索") },
            leadingIcon = { Icon(Icons.Default.Search, null) },
            modifier = Modifier.fillMaxWidth()
        )
        LazyColumn {
            items(results) { item -> ListItem(headlineContent = { Text(item.title) }) }
        }
    }
}
```

---

## throttleFirst

```kotlin
fun <T> Flow<T>.throttleFirst(windowMs: Long): Flow<T> = flow {
    var lastEmit = 0L
    collect { value ->
        val now = System.currentTimeMillis()
        if (now - lastEmit >= windowMs) {
            lastEmit = now
            emit(value)
        }
    }
}

// ボタン連打防止
@Composable
fun ThrottledButton() {
    val clicks = remember { MutableSharedFlow<Unit>(extraBufferCapacity = 1) }
    val scope = rememberCoroutineScope()

    LaunchedEffect(Unit) {
        clicks.throttleFirst(1000).collect {
            // API呼び出し等
        }
    }

    Button(onClick = { scope.launch { clicks.emit(Unit) } }) {
        Text("送信")
    }
}
```

---

## sample

```kotlin
// 一定間隔で最新値を取得
@OptIn(FlowPreview::class)
fun locationUpdates(): Flow<Location> = callbackFlow {
    // 位置情報更新のコールバック
    // ...
}.sample(1000) // 1秒ごとに最新の位置を取得

// センサーデータの間引き
fun sensorFlow(): Flow<SensorData> = sensorUpdates()
    .sample(100)  // 100msごとにサンプリング
    .map { it.toDisplayData() }
```

---

## まとめ

| オペレータ | 用途 |
|-----------|------|
| `debounce` | 入力停止後にemit |
| `throttleFirst` | 期間内の最初のみ |
| `sample` | 一定間隔で最新値 |
| `distinctUntilChanged` | 重複排除 |

- `debounce`で入力が止まるまで待ってから処理
- `throttleFirst`でボタン連打防止
- `distinctUntilChanged`で同じ値の再処理回避
- `flatMapLatest`で前回の検索をキャンセル

---

8種類のAndroidアプリテンプレート（検索機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow combine/zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-combine-2026)
- [Flow retry](https://zenn.dev/myougatheaxo/articles/android-compose-flow-retry-2026)
- [SearchBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-search-bar-2026)
