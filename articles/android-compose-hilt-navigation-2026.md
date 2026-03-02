---
title: "Hilt Navigation完全ガイド — hiltViewModel/NavGraph/スコープ管理"
emoji: "🧭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "hilt"]
published: true
---

## この記事で学べること

**Hilt Navigation**（hiltViewModel、NavGraph内DI、ViewModelスコープ、SharedViewModel）を解説します。

---

## hiltViewModel基本

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    val items = repository.getItems()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}

@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel(),
    onItemClick: (Long) -> Unit
) {
    val items by viewModel.items.collectAsStateWithLifecycle()

    LazyColumn {
        items(items) { item ->
            ListItem(
                headlineContent = { Text(item.title) },
                modifier = Modifier.clickable { onItemClick(item.id) }
            )
        }
    }
}
```

---

## NavHost統合

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onItemClick = { navController.navigate("detail/$it") })
        }
        composable(
            "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.LongType })
        ) {
            DetailScreen(onBack = { navController.popBackStack() })
        }
        navigation(startDestination = "login", route = "auth") {
            composable("login") {
                // authグラフ内で共有ViewModel
                val parentEntry = remember(it) { navController.getBackStackEntry("auth") }
                val authViewModel: AuthViewModel = hiltViewModel(parentEntry)
                LoginScreen(authViewModel)
            }
            composable("register") {
                val parentEntry = remember(it) { navController.getBackStackEntry("auth") }
                val authViewModel: AuthViewModel = hiltViewModel(parentEntry)
                RegisterScreen(authViewModel)
            }
        }
    }
}
```

---

## SavedStateHandle

```kotlin
@HiltViewModel
class DetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: ItemRepository
) : ViewModel() {
    private val itemId: Long = checkNotNull(savedStateHandle["itemId"])

    val item = repository.getItem(itemId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}

@Composable
fun DetailScreen(
    viewModel: DetailViewModel = hiltViewModel(),
    onBack: () -> Unit
) {
    val item by viewModel.item.collectAsStateWithLifecycle()

    item?.let {
        Column(Modifier.padding(16.dp)) {
            Text(it.title, style = MaterialTheme.typography.headlineMedium)
            Text(it.description)
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `hiltViewModel()` | DI対応ViewModel取得 |
| `SavedStateHandle` | ナビゲーション引数 |
| `getBackStackEntry` | 共有ViewModel |
| `navigation` | ネストNavGraph |

- `hiltViewModel()`でComposable内からDI対応ViewModel
- `SavedStateHandle`でナビゲーション引数を自動注入
- `getBackStackEntry`で親NavGraphのViewModelを共有
- ネストNavGraphでスコープ付きViewModel

---

8種類のAndroidアプリテンプレート（Hilt対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Type-safe Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-type-safe-2026)
- [ViewModel SavedState](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-savedstate-2026)
- [Hilt Multimodule](https://zenn.dev/myougatheaxo/articles/android-compose-dagger-hilt-multimodule-2026)
