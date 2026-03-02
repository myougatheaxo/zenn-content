---
title: "sealed interface活用ガイド — 型安全な状態・イベント・ナビゲーション管理"
emoji: "🔒"
type: "tech"
topics: ["android", "kotlin", "sealed", "architecture"]
published: true
---

## この記事で学べること

Kotlinの**sealed interface**を使った型安全な状態管理・イベント処理・ナビゲーションを解説します。

---

## sealed class vs sealed interface

```kotlin
// sealed class: 共通プロパティ/メソッドが必要な場合
sealed class Result<out T>(val timestamp: Long = System.currentTimeMillis()) {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

// sealed interface: 多重継承が必要な場合、より軽量
sealed interface UiEvent
sealed interface NavigationEvent

data class ShowSnackbar(val message: String) : UiEvent
data class NavigateTo(val route: String) : UiEvent, NavigationEvent
data object GoBack : UiEvent, NavigationEvent
```

---

## UI State管理

```kotlin
sealed interface HomeUiState {
    data object Loading : HomeUiState
    data class Success(
        val items: List<Item>,
        val isRefreshing: Boolean = false
    ) : HomeUiState
    data class Error(val message: String) : HomeUiState
}

class HomeViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<HomeUiState>(HomeUiState.Loading)
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    fun loadItems() {
        viewModelScope.launch {
            _uiState.value = HomeUiState.Loading
            repository.getItems().fold(
                onSuccess = { _uiState.value = HomeUiState.Success(it) },
                onFailure = { _uiState.value = HomeUiState.Error(it.message ?: "エラー") }
            )
        }
    }
}

@Composable
fun HomeScreen(viewModel: HomeViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is HomeUiState.Loading -> LoadingIndicator()
        is HomeUiState.Success -> ItemList(state.items)
        is HomeUiState.Error -> ErrorView(state.message) { viewModel.loadItems() }
    }
}
```

---

## ユーザーアクション

```kotlin
sealed interface HomeAction {
    data class Search(val query: String) : HomeAction
    data class SelectItem(val id: String) : HomeAction
    data object Refresh : HomeAction
    data object LoadMore : HomeAction
    data class DeleteItem(val id: String) : HomeAction
}

class HomeViewModel : ViewModel() {
    fun onAction(action: HomeAction) {
        when (action) {
            is HomeAction.Search -> search(action.query)
            is HomeAction.SelectItem -> selectItem(action.id)
            is HomeAction.Refresh -> refresh()
            is HomeAction.LoadMore -> loadMore()
            is HomeAction.DeleteItem -> deleteItem(action.id)
        }
    }
}

@Composable
fun HomeScreen(viewModel: HomeViewModel) {
    // UIからアクション発行
    SearchBar(onSearch = { viewModel.onAction(HomeAction.Search(it)) })
    ItemList(
        onItemClick = { viewModel.onAction(HomeAction.SelectItem(it)) },
        onRefresh = { viewModel.onAction(HomeAction.Refresh) }
    )
}
```

---

## Navigation Routes

```kotlin
sealed interface Screen {
    val route: String

    data object Home : Screen { override val route = "home" }
    data object Settings : Screen { override val route = "settings" }
    data class Detail(val id: String) : Screen {
        override val route = "detail/$id"
        companion object {
            const val ROUTE_PATTERN = "detail/{id}"
        }
    }
    data class Edit(val id: String) : Screen {
        override val route = "edit/$id"
        companion object {
            const val ROUTE_PATTERN = "edit/{id}"
        }
    }
}

// NavHost
NavHost(navController, startDestination = Screen.Home.route) {
    composable(Screen.Home.route) { HomeScreen() }
    composable(Screen.Settings.route) { SettingsScreen() }
    composable(Screen.Detail.ROUTE_PATTERN) { backStackEntry ->
        val id = backStackEntry.arguments?.getString("id") ?: ""
        DetailScreen(id)
    }
}

// 型安全なナビゲーション
fun navigate(screen: Screen) {
    navController.navigate(screen.route)
}
```

---

## エラーハンドリング

```kotlin
sealed interface AppError {
    val message: String

    data class Network(override val message: String = "ネットワークエラー") : AppError
    data class Server(val code: Int, override val message: String) : AppError
    data class Auth(override val message: String = "認証エラー") : AppError
    data class Unknown(override val message: String = "不明なエラー") : AppError
}

fun Throwable.toAppError(): AppError = when (this) {
    is IOException -> AppError.Network()
    is HttpException -> when (code()) {
        401 -> AppError.Auth()
        in 500..599 -> AppError.Server(code(), message())
        else -> AppError.Unknown(message())
    }
    else -> AppError.Unknown(message ?: "不明なエラー")
}
```

---

## まとめ

- `sealed interface`は多重実装可能でメモリ効率が良い
- UI State: Loading/Success/Error パターン
- User Action: sealed interfaceでアクション一覧を型安全に定義
- Navigation: routeプロパティで画面定義
- Error: 種類別のエラー処理を`when`で網羅
- `when`の網羅性チェックでパターン漏れを防止

---

8種類のAndroidアプリテンプレート（型安全なアーキテクチャ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-navigation-basic-2026)
- [Kotlin data classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-2026)
