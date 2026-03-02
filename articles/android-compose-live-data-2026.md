---
title: "LiveData→Flow移行完全ガイド — observeAsState/StateFlow変換/移行パターン"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "migration"]
published: true
---

## この記事で学べること

**LiveData→Flow移行**（observeAsState、StateFlow変換、移行パターン、共存方法）を解説します。

---

## LiveData + Compose（既存コード）

```kotlin
// 既存: LiveDataベースのViewModel
class LegacyViewModel : ViewModel() {
    private val _items = MutableLiveData<List<Item>>(emptyList())
    val items: LiveData<List<Item>> = _items

    fun loadItems() {
        viewModelScope.launch {
            _items.value = repository.getItems()
        }
    }
}

// Composeで使用
@Composable
fun LegacyScreen(viewModel: LegacyViewModel) {
    val items by viewModel.items.observeAsState(emptyList())

    LazyColumn {
        items(items) { item -> ItemRow(item) }
    }
}
```

---

## Flow移行後

```kotlin
// 移行後: StateFlowベース
class ModernViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items.asStateFlow()

    init { loadItems() }

    private fun loadItems() {
        viewModelScope.launch {
            repository.getItems().collect { _items.value = it }
        }
    }
}

// Composeで使用
@Composable
fun ModernScreen(viewModel: ModernViewModel = hiltViewModel()) {
    val items by viewModel.items.collectAsStateWithLifecycle()

    LazyColumn {
        items(items) { item -> ItemRow(item) }
    }
}
```

---

## 移行ヘルパー

```kotlin
// LiveData → Flow変換
fun <T> LiveData<T>.asFlow(): Flow<T> = callbackFlow {
    val observer = Observer<T> { value -> trySend(value) }
    observeForever(observer)
    awaitClose { removeObserver(observer) }
}

// Flow → LiveData変換（段階移行用）
class HybridViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    // 内部はFlow
    private val _itemsFlow = MutableStateFlow<List<Item>>(emptyList())

    // 既存のXMLレイアウト用
    val itemsLiveData: LiveData<List<Item>> = _itemsFlow.asLiveData()

    // Compose用
    val items: StateFlow<List<Item>> = _itemsFlow.asStateFlow()
}
```

---

## まとめ

| 比較 | LiveData | StateFlow |
|------|----------|-----------|
| ライフサイクル | 自動管理 | `collectAsStateWithLifecycle` |
| 初期値 | nullable | 必須 |
| 変換 | `Transformations` | `map`/`combine` |
| テスト | `getOrAwaitValue` | `first()` |

- `observeAsState` → `collectAsStateWithLifecycle`に移行
- `MutableLiveData` → `MutableStateFlow`に変換
- `asLiveData()`で段階的移行（Flow↔LiveData共存）
- 新規コードは`StateFlow`を使用

---

8種類のAndroidアプリテンプレート（StateFlow設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [Coroutine Channel](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-channel-2026)
- [ViewModel Testing](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-testing-2026)
