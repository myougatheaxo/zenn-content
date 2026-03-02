---
title: "Coroutine Flow combine完全ガイド — 複数Flow合成/zip/merge/flatMapLatest"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "flow"]
published: true
---

## この記事で学べること

**Coroutine Flow combine**（combine、zip、merge、flatMapLatest、複数Flowの合成パターン）を解説します。

---

## 基本combine

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {

    private val _query = MutableStateFlow("")
    private val _sortOrder = MutableStateFlow(SortOrder.NAME)
    private val _items = repository.getAllItems()

    // 3つのFlowを合成
    val filteredItems: StateFlow<List<Item>> = combine(
        _query, _sortOrder, _items
    ) { query, sort, items ->
        items.filter { it.name.contains(query, ignoreCase = true) }
            .sortedWith(when (sort) {
                SortOrder.NAME -> compareBy { it.name }
                SortOrder.DATE -> compareByDescending { it.createdAt }
                SortOrder.PRICE -> compareBy { it.price }
            })
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun updateQuery(query: String) { _query.value = query }
    fun updateSort(order: SortOrder) { _sortOrder.value = order }
}

enum class SortOrder { NAME, DATE, PRICE }
```

---

## zip（1対1ペアリング）

```kotlin
fun fetchUserWithProfile(): Flow<Pair<User, Profile>> =
    userFlow.zip(profileFlow) { user, profile ->
        user to profile
    }

// 使用例：2つのAPIを並列呼び出し
suspend fun loadDashboard(): Dashboard {
    val user = async { api.getUser() }
    val stats = async { api.getStats() }
    return Dashboard(user.await(), stats.await())
}
```

---

## flatMapLatest

```kotlin
@HiltViewModel
class CategoryViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {

    private val _selectedCategory = MutableStateFlow<String?>(null)

    // カテゴリ変更のたびに新しいFlowに切り替え
    val items: StateFlow<List<Item>> = _selectedCategory
        .flatMapLatest { category ->
            if (category == null) repository.getAllItems()
            else repository.getItemsByCategory(category)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun selectCategory(category: String?) {
        _selectedCategory.value = category
    }
}
```

---

## merge（複数ソース統合）

```kotlin
fun allNotifications(): Flow<Notification> = merge(
    pushNotifications(),
    localReminders(),
    systemAlerts()
)

@Composable
fun NotificationList(viewModel: NotificationViewModel = hiltViewModel()) {
    val notifications by viewModel.allNotifications.collectAsStateWithLifecycle()

    LazyColumn {
        items(notifications) { notification ->
            ListItem(
                headlineContent = { Text(notification.title) },
                supportingContent = { Text(notification.message) }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `combine` | 最新値の組み合わせ |
| `zip` | 1対1ペアリング |
| `merge` | 複数ソース統合 |
| `flatMapLatest` | Flow切り替え |

- `combine`は任意のFlowが更新されると再計算
- `zip`は両方のFlowが値を出すまで待機
- `flatMapLatest`で前のFlowをキャンセルして新しいFlowに切替
- `merge`で複数ソースを1つのFlowに統合

---

8種類のAndroidアプリテンプレート（リアクティブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow StateFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-state-flow-2026)
- [Flow SharedFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-shared-flow-2026)
- [Flow Debounce](https://zenn.dev/myougatheaxo/articles/android-compose-flow-debounce-2026)
