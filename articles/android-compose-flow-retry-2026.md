---
title: "Flow retry完全ガイド — リトライ戦略/exponential backoff/エラー回復"
emoji: "🔁"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutines"]
published: true
---

## この記事で学べること

**Flow retry**（retry、retryWhen、exponential backoff、エラー回復パターン）を解説します。

---

## retry基本

```kotlin
// シンプルリトライ
fun fetchData(): Flow<List<Item>> = flow {
    emit(api.getItems())
}.retry(3) { cause ->
    cause is IOException  // IOExceptionのみリトライ
}

// retryWhenでカスタムロジック
fun fetchWithBackoff(): Flow<List<Item>> = flow {
    emit(api.getItems())
}.retryWhen { cause, attempt ->
    if (cause is IOException && attempt < 3) {
        delay(1000L * (attempt + 1))  // 1s, 2s, 3s
        true
    } else false
}
```

---

## Exponential Backoff

```kotlin
fun <T> Flow<T>.retryWithExponentialBackoff(
    maxRetries: Int = 3,
    initialDelay: Long = 1000,
    maxDelay: Long = 30000,
    factor: Double = 2.0
): Flow<T> = retryWhen { cause, attempt ->
    if (cause is IOException && attempt < maxRetries) {
        val delay = (initialDelay * factor.pow(attempt.toInt())).toLong()
            .coerceAtMost(maxDelay)
        delay(delay)
        true
    } else false
}

// 使用例
class ItemRepository @Inject constructor(private val api: ItemApi) {
    fun getItems(): Flow<List<Item>> = flow {
        emit(api.getItems())
    }.retryWithExponentialBackoff(maxRetries = 5)
}
```

---

## ViewModel統合

```kotlin
@HiltViewModel
class DataViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState = _uiState.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            repository.getItems()
                .retryWhen { cause, attempt ->
                    _uiState.value = UiState.Retrying(attempt.toInt() + 1)
                    cause is IOException && attempt < 3 && run { delay(2000); true }
                }
                .catch { _uiState.value = UiState.Error(it.message ?: "エラー") }
                .collect { _uiState.value = UiState.Success(it) }
        }
    }
}

sealed interface UiState {
    data object Loading : UiState
    data class Retrying(val attempt: Int) : UiState
    data class Success(val items: List<Item>) : UiState
    data class Error(val message: String) : UiState
}

@Composable
fun DataScreen(viewModel: DataViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    when (val s = state) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Retrying -> Text("リトライ中... (${s.attempt}回目)")
        is UiState.Success -> LazyColumn { items(s.items) { ListItem(headlineContent = { Text(it.title) }) } }
        is UiState.Error -> {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Text(s.message)
                Button(onClick = { viewModel.loadData() }) { Text("再試行") }
            }
        }
    }
}
```

---

## まとめ

| オペレータ | 用途 |
|-----------|------|
| `retry` | 固定回数リトライ |
| `retryWhen` | 条件付きリトライ |
| `catch` | エラーハンドリング |
| Exponential Backoff | 指数的遅延 |

- `retry`でシンプルなリトライ
- `retryWhen`で遅延やエラー種別によるカスタムリトライ
- Exponential Backoffでサーバー負荷を軽減
- UIに「リトライ中」の状態を表示

---

8種類のAndroidアプリテンプレート（エラー処理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Flow debounce](https://zenn.dev/myougatheaxo/articles/android-compose-flow-debounce-2026)
- [Flow combine/zip](https://zenn.dev/myougatheaxo/articles/android-compose-flow-combine-2026)
- [SupervisorJob](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-supervisor-2026)
