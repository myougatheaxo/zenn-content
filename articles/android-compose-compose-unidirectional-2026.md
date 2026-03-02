---
title: "Compose 単方向データフロー完全ガイド — StateHoisting/EventUp/StateDown"
emoji: "⬇️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**Compose 単方向データフロー**（State Hoisting、Event Up/State Down、UiState設計パターン）を解説します。

---

## State Hoisting

```kotlin
// ❌ 内部State: テスト・プレビュー困難
@Composable
fun BadCounter() {
    var count by remember { mutableIntStateOf(0) }
    Button(onClick = { count++ }) { Text("Count: $count") }
}

// ✅ State Hoisting: 状態を上位に持ち上げ
@Composable
fun GoodCounter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("Count: $count") }
}

// 親で状態管理
@Composable
fun CounterScreen() {
    var count by remember { mutableIntStateOf(0) }
    GoodCounter(count = count, onIncrement = { count++ })
}
```

---

## UiState設計

```kotlin
// sealed interfaceでUI状態を型安全に
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

@HiltViewModel
class ItemListViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    val uiState: StateFlow<UiState<List<Item>>> = repository.getItems()
        .map<List<Item>, UiState<List<Item>>> { UiState.Success(it) }
        .catch { emit(UiState.Error(it.message ?: "エラー")) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UiState.Loading)
}

@Composable
fun ItemListScreen(viewModel: ItemListViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        UiState.Loading -> Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
        is UiState.Success -> LazyColumn {
            items(state.data) { ListItem(headlineContent = { Text(it.name) }) }
        }
        is UiState.Error -> Column(Modifier.fillMaxSize(),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center) {
            Text(state.message)
            Button(onClick = { /* retry */ }) { Text("再試行") }
        }
    }
}
```

---

## Event Up / State Down

```kotlin
// 複数コンポーネントの連携
@Composable
fun SearchableList(
    // State Down（親→子）
    query: String,
    items: List<Item>,
    isLoading: Boolean,
    // Event Up（子→親）
    onQueryChange: (String) -> Unit,
    onItemClick: (Item) -> Unit,
    onRefresh: () -> Unit
) {
    Column {
        OutlinedTextField(
            value = query,
            onValueChange = onQueryChange,
            label = { Text("検索") },
            modifier = Modifier.fillMaxWidth().padding(16.dp)
        )

        if (isLoading) {
            LinearProgressIndicator(Modifier.fillMaxWidth())
        }

        LazyColumn {
            items(items) { item ->
                ListItem(
                    headlineContent = { Text(item.name) },
                    modifier = Modifier.clickable { onItemClick(item) }
                )
            }
        }
    }
}
```

---

## まとめ

| 原則 | 説明 |
|------|------|
| State Hoisting | 状態を上位に持ち上げ |
| Event Up | イベントは子→親 |
| State Down | 状態は親→子 |
| UiState | 状態の型安全表現 |

- State Hoistingでテスタビリティ・再利用性向上
- `sealed interface`でUIの全状態を型安全に表現
- イベントはコールバック(ラムダ)で親に伝搬
- 状態はパラメータで子に渡す（単方向）

---

8種類のAndroidアプリテンプレート（アーキテクチャ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MVI](https://zenn.dev/myougatheaxo/articles/android-compose-compose-mvi-2026)
- [Compose CleanArchitecture](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clean-architecture-2026)
- [Flow StateFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-state-flow-2026)
