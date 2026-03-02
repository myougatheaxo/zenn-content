---
title: "Kotlin Channel完全ガイド — Flow vs Channel使い分け"
emoji: "📡"
type: "tech"
topics: ["android", "kotlin", "coroutine", "channel"]
published: true
---

## この記事で学べること

Kotlinの**Channel**（send/receive、BufferOverflow、callbackFlow、Flow vs Channel比較）を解説します。

---

## Channel基礎

```kotlin
// Channelは1対1のデータストリーム
val channel = Channel<Int>()

// 送信側
launch {
    for (i in 1..5) {
        channel.send(i)
        delay(100)
    }
    channel.close()
}

// 受信側
launch {
    for (value in channel) {
        println("Received: $value")
    }
}
```

---

## Channel種別

```kotlin
// Rendezvous（デフォルト）: バッファなし、送受信が同期
val rendezvous = Channel<Int>()

// Buffered: 指定サイズのバッファ
val buffered = Channel<Int>(capacity = 10)

// Unlimited: 無制限バッファ（メモリ注意）
val unlimited = Channel<Int>(Channel.UNLIMITED)

// Conflated: 最新値のみ保持
val conflated = Channel<Int>(Channel.CONFLATED)

// BufferOverflow指定
val dropping = Channel<Int>(
    capacity = 5,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
```

---

## callbackFlow

```kotlin
// コールバックAPIをFlowに変換
fun observeLocation(context: Context): Flow<Location> = callbackFlow {
    val client = LocationServices.getFusedLocationProviderClient(context)

    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.lastLocation?.let { trySend(it) }
        }
    }

    val request = LocationRequest.Builder(
        Priority.PRIORITY_HIGH_ACCURACY, 5000L
    ).build()

    client.requestLocationUpdates(request, callback, Looper.getMainLooper())

    awaitClose {
        client.removeLocationUpdates(callback)
    }
}

// 使用
viewModelScope.launch {
    observeLocation(context)
        .distinctUntilChanged()
        .collect { location ->
            _location.value = location
        }
}
```

---

## イベントバス（SharedFlow）

```kotlin
// 1回限りのイベント通知
class EventBus {
    private val _events = MutableSharedFlow<UiEvent>(
        extraBufferCapacity = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun emit(event: UiEvent) {
        _events.tryEmit(event)
    }
}

sealed interface UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent
    data class Navigate(val route: String) : UiEvent
    data object GoBack : UiEvent
}

// ViewModel
@HiltViewModel
class MainViewModel @Inject constructor() : ViewModel() {
    private val _events = Channel<UiEvent>(Channel.BUFFERED)
    val events = _events.receiveAsFlow()

    fun showMessage(msg: String) {
        viewModelScope.launch {
            _events.send(UiEvent.ShowSnackbar(msg))
        }
    }
}

// Compose
@Composable
fun MainScreen(viewModel: MainViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is UiEvent.Navigate -> { /* navController.navigate */ }
                UiEvent.GoBack -> { /* navController.popBackStack */ }
            }
        }
    }
}
```

---

## Flow vs Channel

```kotlin
// Flow: コールドストリーム（collect時に開始）
val userFlow: Flow<List<User>> = flow {
    val users = repository.getUsers()
    emit(users)
}

// StateFlow: ホットストリーム（常に最新値を保持）
val _users = MutableStateFlow<List<User>>(emptyList())
val users: StateFlow<List<User>> = _users.asStateFlow()

// SharedFlow: ホットストリーム（イベント向け）
val _events = MutableSharedFlow<Event>()

// Channel: 1回限りの消費（イベント/コマンド）
val _actions = Channel<Action>(Channel.BUFFERED)
```

---

## まとめ

| 型 | 性質 | 用途 |
|----|------|------|
| `Flow` | コールド、多対多 | データストリーム |
| `StateFlow` | ホット、最新値保持 | UI状態 |
| `SharedFlow` | ホット、多対多 | イベント通知 |
| `Channel` | ホット、1対1消費 | コマンド/アクション |
| `callbackFlow` | コールバック→Flow | API変換 |

- `callbackFlow` + `awaitClose`でリソース管理
- `Channel.BUFFERED`で1回限りイベント
- `receiveAsFlow()`でChannel→Flow変換
- UIイベントにはChannel、状態にはStateFlow

---

8種類のAndroidアプリテンプレート（Flow/Channel設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow演算子ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-flow-operators-2026)
- [コルーチン基礎](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-basics-2026)
- [Flow+Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
