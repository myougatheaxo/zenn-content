---
title: "Coroutine Actor完全ガイド — メッセージパッシング/Sequential処理/イベント集約"
emoji: "🎭"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**Coroutine Actor**（メッセージパッシング、Sequential処理、Channel+coroutineScope）を解説します。

---

## Actor風パターン

```kotlin
sealed interface CounterMsg {
    data object Increment : CounterMsg
    data object Decrement : CounterMsg
    data class GetValue(val response: CompletableDeferred<Int>) : CounterMsg
}

fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var count = 0
    for (msg in channel) {
        when (msg) {
            CounterMsg.Increment -> count++
            CounterMsg.Decrement -> count--
            is CounterMsg.GetValue -> msg.response.complete(count)
        }
    }
}

suspend fun main() = coroutineScope {
    val counter = counterActor()

    (1..1000).map { launch { counter.send(CounterMsg.Increment) } }.joinAll()

    val result = CompletableDeferred<Int>()
    counter.send(CounterMsg.GetValue(result))
    println(result.await()) // 1000

    counter.close()
}
```

---

## UIイベント集約

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    private val events = Channel<SearchEvent>(Channel.BUFFERED)
    val results = MutableStateFlow<List<SearchResult>>(emptyList())

    init {
        viewModelScope.launch {
            events.receiveAsFlow()
                .debounce(300)
                .distinctUntilChanged()
                .collectLatest { event ->
                    when (event) {
                        is SearchEvent.Query -> {
                            results.value = repository.search(event.text)
                        }
                        is SearchEvent.Clear -> {
                            results.value = emptyList()
                        }
                    }
                }
        }
    }

    fun onEvent(event: SearchEvent) {
        viewModelScope.launch { events.send(event) }
    }
}

sealed interface SearchEvent {
    data class Query(val text: String) : SearchEvent
    data object Clear : SearchEvent
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `actor` | メッセージ駆動処理 |
| `Channel` | イベントキュー |
| `CompletableDeferred` | 結果受け取り |
| `receiveAsFlow` | Channel→Flow |

- Actor風パターンで状態を安全にカプセル化
- Channelでイベントをシリアライズ処理
- ViewModelでUIイベントの集約に活用
- `debounce` + `distinctUntilChanged`で最適化

---

8種類のAndroidアプリテンプレート（非同期処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Channel](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-channel-2026)
- [Mutex](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-mutex-2026)
- [Flow debounce](https://zenn.dev/myougatheaxo/articles/android-compose-flow-debounce-2026)
