---
title: "Kotlin Coroutines & Flow入門 — 非同期処理の正しいパターン"
emoji: "🌊"
type: "tech"
topics: ["android", "kotlin", "coroutines", "flow"]
published: true
---

## この記事で学べること

Androidの非同期処理は**Coroutines + Flow**が標準です。コールバック地獄から脱出し、読みやすい非同期コードを書く方法を解説します。

---

## suspend関数の基本

```kotlin
// ❌ メインスレッドでネットワーク通信（ANR）
fun loadData() {
    val result = api.getUsers() // ブロッキング
}

// ✅ suspend関数で非同期
suspend fun loadData(): List<User> {
    return withContext(Dispatchers.IO) {
        api.getUsers()
    }
}
```

`suspend`関数はCoroutineの中でしか呼べません。重い処理は`Dispatchers.IO`で実行。

---

## ViewModelでのCoroutine

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()

    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _isLoading.value = true
            try {
                _users.value = repository.getUsers()
            } catch (e: Exception) {
                // エラー処理
            } finally {
                _isLoading.value = false
            }
        }
    }
}
```

`viewModelScope`はViewModelのライフサイクルに連動。画面を離れると自動キャンセル。

---

## Flow（データストリーム）

```kotlin
// Repository
class UserRepository(private val dao: UserDao) {
    fun observeUsers(): Flow<List<User>> = dao.getAllUsers()
}

// DAO
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY name")
    fun getAllUsers(): Flow<List<User>>
}
```

RoomのDAOが`Flow`を返せば、**DBの変更が自動でUIに反映**されます。

---

## ComposeでFlowを購読

```kotlin
@Composable
fun UserListScreen(viewModel: UserViewModel) {
    val users by viewModel.users.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()

    if (isLoading) {
        CircularProgressIndicator()
    } else {
        LazyColumn {
            items(users, key = { it.id }) { user ->
                Text(user.name)
            }
        }
    }
}
```

`collectAsState()`でFlowをComposeのStateに変換。

---

## Flowの変換オペレータ

```kotlin
class SearchViewModel(private val repo: UserRepository) : ViewModel() {
    private val _query = MutableStateFlow("")

    val searchResults: StateFlow<List<User>> = _query
        .debounce(300)                    // 300ms入力待ち
        .filter { it.length >= 2 }        // 2文字以上
        .flatMapLatest { query ->         // 最新のクエリだけ実行
            repo.searchUsers(query)
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun onQueryChanged(query: String) {
        _query.value = query
    }
}
```

| オペレータ | 用途 |
|-----------|------|
| `map` | 値を変換 |
| `filter` | 条件でフィルタ |
| `debounce` | 一定時間入力がなければ発火 |
| `flatMapLatest` | 前の処理をキャンセルして最新を実行 |
| `combine` | 複数Flowを合成 |
| `catch` | エラーハンドリング |

---

## stateIn / shareIn

```kotlin
// Cold Flow → Hot StateFlow
val users: StateFlow<List<User>> = repo.observeUsers()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyList()
    )
```

| パラメータ | 意味 |
|-----------|------|
| `WhileSubscribed(5000)` | 購読者がいなくなって5秒後に停止 |
| `Lazily` | 最初の購読者が来たら開始 |
| `Eagerly` | 即座に開始 |

`WhileSubscribed(5000)`が最も推奨。画面回転（~1秒）では停止せず、画面離脱では停止。

---

## エラーハンドリング

```kotlin
viewModelScope.launch {
    repo.observeUsers()
        .catch { e ->
            _error.value = e.message
        }
        .collect { users ->
            _users.value = users
        }
}
```

---

## まとめ

| 概念 | 用途 |
|------|------|
| `suspend` | 単発の非同期処理 |
| `Flow` | データストリーム（連続的な値） |
| `StateFlow` | UIに公開するHot Flow |
| `viewModelScope` | ViewModel連動のCoroutineスコープ |
| `collectAsState()` | ComposeでFlowを購読 |
| `stateIn` | Cold→Hotの変換 |

---

8種類のAndroidアプリテンプレート（Coroutines + Flow正しく実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Retrofit入門](https://zenn.dev/myougatheaxo/articles/android-retrofit-api-2026)
- [Composeの状態管理入門](https://zenn.dev/myougatheaxo/articles/compose-state-management-2026)
