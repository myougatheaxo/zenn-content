---
title: "Flow/StateFlow実践パターン集 — combine/flatMap/debounce/retry"
emoji: "🌊"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "coroutines"]
published: true
---

## この記事で学べること

**Flow/StateFlow実践パターン**（combine、flatMapLatest、debounce、retry、callbackFlow、テスト）を解説します。

---

## combine: 複数Flowの結合

```kotlin
@HiltViewModel
class DashboardViewModel @Inject constructor(
    userRepository: UserRepository,
    orderRepository: OrderRepository,
    notificationRepository: NotificationRepository
) : ViewModel() {

    val dashboardState = combine(
        userRepository.getUser(),
        orderRepository.getRecentOrders(),
        notificationRepository.getUnreadCount()
    ) { user, orders, unreadCount ->
        DashboardState(user, orders, unreadCount)
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), DashboardState())
}
```

---

## flatMapLatest: 検索

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    private val _query = MutableStateFlow("")

    val results = _query
        .debounce(300)  // 300ms待ってから検索
        .distinctUntilChanged()
        .flatMapLatest { query ->
            if (query.isBlank()) flowOf(emptyList())
            else repository.search(query)
        }
        .catch { emit(emptyList()) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onQueryChange(query: String) { _query.value = query }
}
```

---

## retry: エラーリトライ

```kotlin
fun getDataWithRetry(): Flow<List<Item>> = flow {
    emit(api.getItems())
}.retry(3) { cause ->
    cause is IOException && run {
        delay(1000)
        true
    }
}.catch { emit(emptyList()) }

// exponential backoff
fun <T> Flow<T>.retryWithBackoff(
    maxRetries: Int = 3,
    initialDelay: Long = 1000
): Flow<T> = retryWhen { cause, attempt ->
    if (attempt < maxRetries && cause is IOException) {
        delay(initialDelay * (1 shl attempt.toInt()))
        true
    } else false
}
```

---

## callbackFlow: コールバック→Flow変換

```kotlin
fun locationUpdates(context: Context): Flow<Location> = callbackFlow {
    val client = LocationServices.getFusedLocationProviderClient(context)
    val request = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 5000).build()

    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.lastLocation?.let { trySend(it) }
        }
    }

    client.requestLocationUpdates(request, callback, Looper.getMainLooper())
    awaitClose { client.removeLocationUpdates(callback) }
}
```

---

## stateIn vs shareIn

```kotlin
// stateIn: 最新値を保持（UI状態向け）
val uiState = flow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),  // 5秒キャッシュ
    initialValue = UiState.Loading
)

// shareIn: イベントストリーム（通知向け）
val events = flow.shareIn(
    scope = viewModelScope,
    started = SharingStarted.Eagerly,
    replay = 0  // 新しいサブスクライバに過去のイベントを送らない
)
```

---

## テスト（Turbine）

```kotlin
@Test
fun searchReturnsResults() = runTest {
    val viewModel = SearchViewModel(FakeSearchRepository())

    viewModel.results.test {
        assertEquals(emptyList<SearchResult>(), awaitItem()) // 初期値

        viewModel.onQueryChange("kotlin")
        advanceTimeBy(300) // debounce待ち

        val results = awaitItem()
        assertTrue(results.isNotEmpty())
    }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `combine` | 複数Flowを結合 |
| `flatMapLatest` | 最新のみ処理 |
| `debounce` | 入力待ち |
| `retry` | エラーリトライ |
| `callbackFlow` | コールバック変換 |
| `stateIn` | UI状態の共有 |

- `combine`で複数データソースを統合
- `debounce` + `flatMapLatest`で検索最適化
- `callbackFlow`でレガシーAPIをFlow化
- `WhileSubscribed(5000)`でバックグラウンド最適化

---

8種類のAndroidアプリテンプレート（Flow設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Flow](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-2026)
- [Flow演算子](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-operators-2026)
- [Coroutineテスト](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-testing-2026)
