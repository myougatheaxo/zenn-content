---
title: "Coroutines & Flow入門 — Kotlinの非同期処理を完全理解"
emoji: "🌊"
type: "tech"
topics: ["android", "kotlin", "coroutines", "flow"]
published: true
---

## この記事で学べること

Kotlinの**Coroutines**と**Flow**を使った非同期処理の基本から実践パターンまで解説します。

---

## Coroutinesの基本

```kotlin
class MyViewModel : ViewModel() {
    fun fetchData() {
        viewModelScope.launch {
            val result = withContext(Dispatchers.IO) {
                repository.getData() // IO スレッドで実行
            }
            _uiState.value = result // Mainスレッドで更新
        }
    }
}
```

### ディスパッチャー

```kotlin
// UIスレッド
Dispatchers.Main

// バックグラウンド（ネットワーク・DB）
Dispatchers.IO

// CPU集約処理（ソート・パース）
Dispatchers.Default
```

---

## suspend関数

```kotlin
// suspend関数はコルーチン内からのみ呼べる
suspend fun fetchUser(id: String): User {
    return withContext(Dispatchers.IO) {
        api.getUser(id)
    }
}

// 並列実行
suspend fun fetchDashboard(): Dashboard {
    return coroutineScope {
        val user = async { fetchUser("123") }
        val posts = async { fetchPosts("123") }
        Dashboard(user.await(), posts.await())
    }
}
```

---

## Flow の基本

```kotlin
// Flowの作成
fun getItemsFlow(): Flow<List<Item>> = flow {
    while (true) {
        val items = repository.getItems()
        emit(items)
        delay(5000) // 5秒ごとにポーリング
    }
}

// Roomから直接Flow
@Dao
interface ItemDao {
    @Query("SELECT * FROM items ORDER BY createdAt DESC")
    fun observeItems(): Flow<List<Item>>
}
```

---

## StateFlow / SharedFlow

```kotlin
class ItemViewModel : ViewModel() {
    // StateFlow: 常に最新値を保持
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items.asStateFlow()

    // SharedFlow: イベント通知（1回限り）
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun deleteItem(id: String) {
        viewModelScope.launch {
            repository.delete(id)
            _events.emit(UiEvent.ShowSnackbar("削除しました"))
        }
    }
}

sealed class UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
}
```

---

## ComposeでFlowを収集

```kotlin
@Composable
fun ItemScreen(viewModel: ItemViewModel) {
    val items by viewModel.items.collectAsStateWithLifecycle()

    // イベント収集
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is UiEvent.Navigate -> {
                    navController.navigate(event.route)
                }
            }
        }
    }

    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemRow(item)
        }
    }
}
```

---

## Flow演算子

```kotlin
class SearchViewModel : ViewModel() {
    private val _query = MutableStateFlow("")

    val searchResults = _query
        .debounce(300) // 300ms待機
        .distinctUntilChanged() // 重複除去
        .filter { it.length >= 2 } // 2文字以上
        .flatMapLatest { query ->
            repository.search(query) // 最新のみ
        }
        .catch { e ->
            emit(emptyList()) // エラー時は空リスト
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    fun onQueryChange(query: String) {
        _query.value = query
    }
}
```

---

## エラーハンドリング

```kotlin
class DataViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val data = withContext(Dispatchers.IO) {
                    repository.fetchData()
                }
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "不明なエラー")
            }
        }
    }
}

sealed class UiState {
    data object Loading : UiState()
    data class Success(val data: List<Item>) : UiState()
    data class Error(val message: String) : UiState()
}
```

---

## まとめ

- `viewModelScope.launch`でCoroutine起動
- `withContext(Dispatchers.IO)`でIOスレッド切り替え
- `async/await`で並列実行
- `StateFlow`で状態管理、`SharedFlow`でイベント通知
- `collectAsStateWithLifecycle()`でCompose収集
- `debounce`, `distinctUntilChanged`, `flatMapLatest`で検索最適化
- `stateIn`でFlowをStateFlowに変換

---

8種類のAndroidアプリテンプレート（Coroutines/Flow設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [Room Database完全ガイド](https://zenn.dev/myougatheaxo/articles/android-room-database-2026)
- [Hilt依存性注入ガイド](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
