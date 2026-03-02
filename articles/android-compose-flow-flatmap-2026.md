---
title: "Flow flatMap完全ガイド — flatMapConcat/flatMapMerge/flatMapLatest"
emoji: "🔃"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "flow"]
published: true
---

## この記事で学べること

**Flow flatMap**（flatMapConcat、flatMapMerge、flatMapLatest、ネストFlow展開）を解説します。

---

## flatMapConcat（順次実行）

```kotlin
// 各値に対して順番にFlowを実行
val categories = flowOf("tech", "design", "business")

categories.flatMapConcat { category ->
    flow {
        val articles = api.getArticles(category) // 順番に呼ぶ
        articles.forEach { emit(it) }
    }
}.collect { article ->
    println(article.title)
}

// tech → design → business の順で処理
```

---

## flatMapMerge（並行実行）

```kotlin
// 各値に対して並行にFlowを実行
val userIds = flowOf("user1", "user2", "user3")

userIds.flatMapMerge(concurrency = 3) { userId ->
    flow {
        val profile = api.getProfile(userId) // 並行で呼ぶ
        emit(profile)
    }
}
.collect { profile ->
    println(profile.name)
}

// user1, user2, user3 が並行に取得される
```

---

## flatMapLatest（最新のみ）

```kotlin
// 新しい値が来たら前のFlowをキャンセル
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    private val _query = MutableStateFlow("")

    val results: StateFlow<List<SearchResult>> = _query
        .debounce(300)
        .filter { it.length >= 2 }
        .flatMapLatest { query ->
            // 新しいクエリが来たら前の検索をキャンセル
            flow {
                emit(emptyList()) // ローディング表示
                val results = repository.search(query)
                emit(results)
            }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun updateQuery(query: String) { _query.value = query }
}

@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel._query.collectAsState() // 実際はpublicにする
    val results by viewModel.results.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(value = query, onValueChange = { viewModel.updateQuery(it) },
            label = { Text("検索") }, modifier = Modifier.fillMaxWidth())

        LazyColumn {
            items(results) { result ->
                ListItem(headlineContent = { Text(result.title) })
            }
        }
    }
}
```

---

## まとめ

| 演算子 | 動作 |
|--------|------|
| `flatMapConcat` | 順次実行（完了を待つ） |
| `flatMapMerge` | 並行実行 |
| `flatMapLatest` | 最新のみ（前をキャンセル） |

- `flatMapConcat`は順番が重要な時に使用
- `flatMapMerge`は並行処理で高速化したい時に使用
- `flatMapLatest`は検索・フィルタ等の最新値のみ必要な時に使用
- `debounce`と`flatMapLatest`はリアルタイム検索の定番パターン

---

8種類のAndroidアプリテンプレート（Flow対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-zip-2026)
- [Flow Transform](https://zenn.dev/myougatheaxo/articles/android-compose-flow-transform-2026)
- [Flow Debounce](https://zenn.dev/myougatheaxo/articles/android-compose-flow-debounce-2026)
