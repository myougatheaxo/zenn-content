---
title: "StateFlow完全ガイド — UI状態管理/stateIn/WhileSubscribed"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**StateFlow**（MutableStateFlow、stateIn、SharingStarted、Compose連携）を解説します。

---

## StateFlow基本

```kotlin
@HiltViewModel
class CounterViewModel @Inject constructor() : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count = _count.asStateFlow()

    fun increment() { _count.value++ }
    fun decrement() { _count.value-- }
}

@Composable
fun CounterScreen(viewModel: CounterViewModel = hiltViewModel()) {
    val count by viewModel.count.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
        Text("$count", style = MaterialTheme.typography.displayLarge)
        Row {
            Button(onClick = { viewModel.decrement() }) { Text("-") }
            Spacer(Modifier.width(16.dp))
            Button(onClick = { viewModel.increment() }) { Text("+") }
        }
    }
}
```

---

## stateIn変換

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    // Flow→StateFlowに変換
    val users = repository.getUsers()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    // 複数Flowの結合
    private val _searchQuery = MutableStateFlow("")
    val searchQuery = _searchQuery.asStateFlow()

    val filteredUsers = combine(users, _searchQuery) { userList, query ->
        if (query.isEmpty()) userList
        else userList.filter { it.name.contains(query, ignoreCase = true) }
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onSearch(query: String) { _searchQuery.value = query }
}
```

---

## SharingStarted比較

```kotlin
// Eagerly: 即座に開始、終了しない
val eager = flow.stateIn(scope, SharingStarted.Eagerly, default)

// Lazily: 最初のコレクターで開始、終了しない
val lazy = flow.stateIn(scope, SharingStarted.Lazily, default)

// WhileSubscribed: コレクターがいる間のみ
val subscribed = flow.stateIn(
    scope,
    SharingStarted.WhileSubscribed(
        stopTimeoutMillis = 5000,  // 最後のコレクター解除後5秒で停止
        replayExpirationMillis = 0 // 停止時にキャッシュを即クリア
    ),
    default
)
```

---

## まとめ

| API | 用途 |
|-----|------|
| `MutableStateFlow` | 変更可能な状態 |
| `asStateFlow` | 読み取り専用変換 |
| `stateIn` | Flow→StateFlow変換 |
| `WhileSubscribed` | ライフサイクル連動 |

- `MutableStateFlow`でUI状態を管理
- `stateIn`でFlow→StateFlow変換
- `WhileSubscribed(5000)`で画面回転時のリソース保持
- `collectAsStateWithLifecycle`でCompose連携

---

8種類のAndroidアプリテンプレート（状態管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [SharedFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-shared-flow-2026)
- [Flow combine/zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-combine-2026)
- [ViewModel SavedState](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-savedstate-2026)
