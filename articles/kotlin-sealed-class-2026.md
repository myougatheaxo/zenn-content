---
title: "Kotlin sealed class完全ガイド — UIState・Result・ナビゲーションの型安全設計"
emoji: "🎯"
type: "tech"
topics: ["android", "kotlin", "architecture", "sealed"]
published: true
---

## この記事で学べること

`sealed class`はKotlinの最強機能の一つ。**UIの状態管理、API結果の表現、ナビゲーション定義**に使えば、型安全でバグの少ないコードが書けます。

---

## sealed classとは

```kotlin
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

`sealed class`の子クラスは**同じファイル内でしか定義できない**ため、`when`式で全パターンを網羅できます。

---

## UIState管理パターン

```kotlin
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<User>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<User>>> = _uiState.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val users = repo.getUsers()
                _uiState.value = UiState.Success(users)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "エラーが発生しました")
            }
        }
    }
}
```

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is UiState.Loading -> {
            CircularProgressIndicator()
        }
        is UiState.Success -> {
            val users = (uiState as UiState.Success).data
            LazyColumn {
                items(users) { user -> UserItem(user) }
            }
        }
        is UiState.Error -> {
            val message = (uiState as UiState.Error).message
            Text("エラー: $message")
        }
    }
}
```

---

## Result型パターン

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val exception: Exception) : Result<Nothing>()
}

// Repository
class UserRepository(private val api: UserApi) {
    suspend fun getUsers(): Result<List<User>> {
        return try {
            val users = api.fetchUsers()
            Result.Success(users)
        } catch (e: Exception) {
            Result.Failure(e)
        }
    }
}

// 使い方
when (val result = repo.getUsers()) {
    is Result.Success -> showUsers(result.data)
    is Result.Failure -> showError(result.exception.message)
}
```

---

## sealed interface（Kotlin 1.5+）

```kotlin
sealed interface Screen {
    data object Home : Screen
    data object Settings : Screen
    data class Detail(val id: Int) : Screen
    data class Profile(val userId: String) : Screen
}

// Navigation
@Composable
fun AppNavigation(currentScreen: Screen) {
    when (currentScreen) {
        Screen.Home -> HomeScreen()
        Screen.Settings -> SettingsScreen()
        is Screen.Detail -> DetailScreen(currentScreen.id)
        is Screen.Profile -> ProfileScreen(currentScreen.userId)
    }
}
```

`sealed interface`は複数インターフェースを実装できる点が`sealed class`との違い。

---

## イベント管理

```kotlin
sealed class UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
    data object NavigateBack : UiEvent()
}

class TaskViewModel : ViewModel() {
    private val _events = Channel<UiEvent>()
    val events = _events.receiveAsFlow()

    fun deleteTask(task: Task) {
        viewModelScope.launch {
            repo.delete(task)
            _events.send(UiEvent.ShowSnackbar("${task.name}を削除しました"))
        }
    }
}
```

---

## whenの網羅性チェック

```kotlin
// ✅ 全パターンを網羅 → コンパイル通る
when (state) {
    is UiState.Loading -> { /* ... */ }
    is UiState.Success -> { /* ... */ }
    is UiState.Error -> { /* ... */ }
}

// ❌ パターン不足 → コンパイルエラー
when (state) {
    is UiState.Loading -> { /* ... */ }
    is UiState.Success -> { /* ... */ }
    // Error が足りない！
}
```

新しい状態を追加したとき、**コンパイルエラーで修正漏れを防げる**のが最大のメリット。

---

## まとめ

| 用途 | sealed class/interface |
|------|----------------------|
| UI状態管理 | `Loading / Success / Error` |
| API結果 | `Result<T>` |
| ナビゲーション | `Screen` |
| UIイベント | `UiEvent` |

- `when`式で**全パターンの網羅性**をコンパイラが保証
- `sealed interface`は複数インターフェース実装可能
- `data object`でシングルトン状態を定義

---

8種類のAndroidアプリテンプレート（sealed classによる型安全設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [MVVM Architecture完全ガイド](https://zenn.dev/myougatheaxo/articles/android-mvvm-architecture-2026)
- [Kotlin基礎10パターン](https://zenn.dev/myougatheaxo/articles/kotlin-basics-for-ai-code-2026)
