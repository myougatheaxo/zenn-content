---
title: "SharedFlow完全ガイド — イベント配信/replay/マルチコレクター"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**SharedFlow**（MutableSharedFlow、replay、イベント配信、マルチコレクター）を解説します。

---

## SharedFlow基本

```kotlin
@HiltViewModel
class EventViewModel @Inject constructor() : ViewModel() {
    // replay=0: 新しいコレクターには過去のイベントを送らない
    private val _events = MutableSharedFlow<UiEvent>()
    val events = _events.asSharedFlow()

    fun showSnackbar(message: String) {
        viewModelScope.launch { _events.emit(UiEvent.ShowSnackbar(message)) }
    }

    fun navigate(route: String) {
        viewModelScope.launch { _events.emit(UiEvent.Navigate(route)) }
    }
}

sealed interface UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent
    data class Navigate(val route: String) : UiEvent
}

@Composable
fun EventScreen(viewModel: EventViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
                is UiEvent.Navigate -> { /* navController.navigate(event.route) */ }
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        Column(Modifier.padding(padding).padding(16.dp)) {
            Button(onClick = { viewModel.showSnackbar("保存しました") }) { Text("保存") }
        }
    }
}
```

---

## replay付きSharedFlow

```kotlin
class ConnectionMonitor @Inject constructor(
    @ApplicationContext context: Context
) {
    // replay=1: 新しいコレクターは最新の接続状態を即取得
    private val _connectionState = MutableSharedFlow<Boolean>(replay = 1)
    val connectionState = _connectionState.asSharedFlow()

    init {
        val cm = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { _connectionState.tryEmit(true) }
            override fun onLost(network: Network) { _connectionState.tryEmit(false) }
        }
        cm.registerDefaultNetworkCallback(callback)
    }
}
```

---

## SharedFlow vs StateFlow

```kotlin
// StateFlow: 常に最新の値を保持、初期値必須
val counter = MutableStateFlow(0)

// SharedFlow: イベント配信向け、値を保持しない（replay=0時）
val events = MutableSharedFlow<UiEvent>()

// StateFlowはSharedFlowの特殊ケース
// StateFlow = SharedFlow(replay=1) + distinctUntilChanged
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `replay = 0` | ワンショットイベント |
| `replay = 1` | 最新値キャッシュ |
| `tryEmit` | ノンサスペンド送信 |
| `extraBufferCapacity` | バッファサイズ |

- `SharedFlow`でワンショットイベント配信
- `replay`で新規コレクターへの過去イベント提供
- `StateFlow`は最新値の保持に、`SharedFlow`はイベントに
- Snackbar/Navigation/Errorイベントに最適

---

8種類のAndroidアプリテンプレート（イベント処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [StateFlow](https://zenn.dev/myougatheaxo/articles/android-compose-flow-state-flow-2026)
- [Channel](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-channel-2026)
- [Flow debounce](https://zenn.dev/myougatheaxo/articles/android-compose-flow-debounce-2026)
