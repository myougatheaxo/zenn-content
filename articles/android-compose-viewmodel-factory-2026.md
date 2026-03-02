---
title: "ViewModel Factory完全ガイド — viewModelFactory/CreationExtras/AssistedInject"
emoji: "🏭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "viewmodel"]
published: true
---

## この記事で学べること

**ViewModel Factory**（viewModelFactory、CreationExtras、AssistedInject）を解説します。

---

## viewModelFactory

```kotlin
// viewModelFactory DSL
val factory = viewModelFactory {
    initializer {
        val application = this[APPLICATION_KEY] as MyApp
        val repository = application.container.userRepository
        UserListViewModel(repository)
    }
}

@Composable
fun UserListScreen() {
    val viewModel: UserListViewModel = viewModel(factory = factory)
    val users by viewModel.users.collectAsStateWithLifecycle()

    LazyColumn {
        items(users) { user ->
            ListItem(headlineContent = { Text(user.name) })
        }
    }
}
```

---

## CreationExtras

```kotlin
class DetailViewModel(
    private val itemId: Long,
    private val repository: ItemRepository
) : ViewModel() {
    val item = repository.getItem(itemId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)

    companion object {
        val ITEM_ID_KEY = object : CreationExtras.Key<Long> {}

        val Factory = viewModelFactory {
            initializer {
                val itemId = this[ITEM_ID_KEY] ?: error("itemId required")
                val app = this[APPLICATION_KEY] as MyApp
                DetailViewModel(itemId, app.container.itemRepository)
            }
        }
    }
}

@Composable
fun DetailScreen(itemId: Long) {
    val viewModel: DetailViewModel = viewModel(
        factory = DetailViewModel.Factory,
        extras = MutableCreationExtras().apply {
            set(DetailViewModel.ITEM_ID_KEY, itemId)
            set(APPLICATION_KEY, LocalContext.current.applicationContext as Application)
        }
    )
    // ...
}
```

---

## Hilt AssistedInject

```kotlin
@HiltViewModel(assistedFactory = DetailViewModel.Factory::class)
class DetailViewModel @AssistedInject constructor(
    @Assisted private val itemId: Long,
    private val repository: ItemRepository
) : ViewModel() {
    val item = repository.getItem(itemId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)

    @AssistedFactory
    interface Factory {
        fun create(itemId: Long): DetailViewModel
    }
}

@Composable
fun DetailScreen(itemId: Long) {
    val viewModel = hiltViewModel<DetailViewModel, DetailViewModel.Factory> { factory ->
        factory.create(itemId)
    }
    // ...
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `viewModelFactory` | Factory DSL |
| `CreationExtras` | 追加パラメータ |
| `@AssistedInject` | Hilt動的引数 |
| `@AssistedFactory` | Factory interface |

- `viewModelFactory`でDSLベースのFactory定義
- `CreationExtras`でViewModelに追加パラメータ
- `@AssistedInject`でHilt+動的引数のViewModel
- Hilt使用時は`hiltViewModel`のFactory版を使用

---

8種類のAndroidアプリテンプレート（ViewModel実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel SavedState](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-savedstate-2026)
- [Hilt/DI](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-2026)
- [Koin + Compose](https://zenn.dev/myougatheaxo/articles/android-compose-koin-compose-2026)
