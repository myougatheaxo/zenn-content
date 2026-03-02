---
title: "Coroutine Dispatcher完全ガイド — Main/IO/Default/Unconfined使い分け"
emoji: "🔀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Coroutine Dispatcher**（Main、IO、Default、Unconfined、カスタムDispatcher）を解説します。

---

## Dispatcher使い分け

```kotlin
// Main: UIスレッド（Compose State更新）
viewModelScope.launch(Dispatchers.Main) {
    _uiState.value = UiState.Loading
}

// IO: ネットワーク/ファイルI/O
suspend fun fetchData(): List<Item> = withContext(Dispatchers.IO) {
    apiService.getItems() // ネットワーク
}

// Default: CPU計算（ソート、パース、暗号化）
suspend fun processData(items: List<Item>): List<Item> = withContext(Dispatchers.Default) {
    items.sortedByDescending { it.score }
        .filter { it.isValid }
        .map { it.transform() }
}

// 組み合わせ例
@HiltViewModel
class DataViewModel @Inject constructor(
    private val repository: DataRepository
) : ViewModel() {
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items

    fun load() {
        viewModelScope.launch { // Main
            val raw = repository.fetchItems()         // IO内部で切替
            val processed = processData(raw)          // Default
            _items.value = processed                  // Main（自動）
        }
    }
}
```

---

## DI経由のDispatcher注入

```kotlin
// テスタビリティのためDI注入
@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default

    @Provides @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}

@Qualifier @Retention(AnnotationRetention.BINARY) annotation class IoDispatcher
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class DefaultDispatcher
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class MainDispatcher

// 使用
class UserRepository @Inject constructor(
    private val api: ApiService,
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) {
    suspend fun getUsers(): List<User> = withContext(ioDispatcher) {
        api.getUsers()
    }
}
```

---

## テスト時のDispatcher差替え

```kotlin
@Test
fun testWithTestDispatcher() = runTest {
    val testDispatcher = UnconfinedTestDispatcher(testScheduler)
    Dispatchers.setMain(testDispatcher)

    try {
        val viewModel = DataViewModel(FakeRepository())
        viewModel.load()

        assertEquals(3, viewModel.items.value.size)
    } finally {
        Dispatchers.resetMain()
    }
}
```

---

## まとめ

| Dispatcher | 用途 |
|------------|------|
| `Main` | UI更新 |
| `IO` | ネットワーク/ファイル |
| `Default` | CPU計算 |
| `Unconfined` | テスト用 |

- `withContext()`で安全にDispatcher切替
- `viewModelScope`はデフォルトで`Dispatchers.Main`
- テストでは`UnconfinedTestDispatcher`で即実行
- DI経由で注入するとテスタビリティ向上

---

8種類のAndroidアプリテンプレート（Coroutine対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Scope](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-scope-2026)
- [Coroutine ExceptionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-handler-2026)
- [Coroutine Cancellation](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-cancellation-2026)
