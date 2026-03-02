---
title: "エラーバウンダリ完全ガイド — Composableクラッシュ防止/フォールバックUI"
emoji: "🛡️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "errorhandling"]
published: true
---

## この記事で学べること

**エラーバウンダリ**（Composableクラッシュ防止、フォールバックUI、グローバル例外ハンドラー、Crashlytics連携）を解説します。

---

## ErrorBoundary Composable

```kotlin
@Composable
fun ErrorBoundary(
    fallback: @Composable (Throwable) -> Unit = { DefaultErrorFallback(it) },
    content: @Composable () -> Unit
) {
    var error by remember { mutableStateOf<Throwable?>(null) }

    if (error != null) {
        fallback(error!!)
    } else {
        try {
            content()
        } catch (e: Exception) {
            error = e
            LaunchedEffect(e) {
                Firebase.crashlytics.recordException(e)
            }
        }
    }
}

@Composable
fun DefaultErrorFallback(error: Throwable) {
    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(Icons.Default.Error, null, Modifier.size(48.dp), tint = MaterialTheme.colorScheme.error)
        Spacer(Modifier.height(8.dp))
        Text("エラーが発生しました", style = MaterialTheme.typography.headlineSmall)
        Text(error.message ?: "", style = MaterialTheme.typography.bodySmall, color = MaterialTheme.colorScheme.onSurfaceVariant)
    }
}
```

---

## 非同期エラーハンドリング

```kotlin
@Composable
fun <T> AsyncContent(
    state: AsyncState<T>,
    onRetry: () -> Unit,
    content: @Composable (T) -> Unit
) {
    when (state) {
        AsyncState.Loading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        is AsyncState.Success -> content(state.data)
        is AsyncState.Error -> {
            Column(
                Modifier.fillMaxSize().padding(16.dp),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Icon(Icons.Default.CloudOff, null, Modifier.size(48.dp))
                Spacer(Modifier.height(8.dp))
                Text(state.message)
                Spacer(Modifier.height(16.dp))
                Button(onClick = onRetry) { Text("再試行") }
            }
        }
    }
}

sealed interface AsyncState<out T> {
    data object Loading : AsyncState<Nothing>
    data class Success<T>(val data: T) : AsyncState<T>
    data class Error(val message: String, val cause: Throwable? = null) : AsyncState<Nothing>
}
```

---

## グローバル例外ハンドラー

```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        val defaultHandler = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
            Firebase.crashlytics.recordException(throwable)
            // カスタム処理（ログ保存等）
            defaultHandler?.uncaughtException(thread, throwable)
        }
    }
}
```

---

## CoroutineExceptionHandler

```kotlin
@HiltViewModel
class MyViewModel @Inject constructor() : ViewModel() {
    private val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
        _uiState.value = UiState.Error(throwable.message ?: "Unknown error")
        Firebase.crashlytics.recordException(throwable)
    }

    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState

    fun loadData() {
        viewModelScope.launch(exceptionHandler) {
            _uiState.value = UiState.Loading
            val data = repository.getData()
            _uiState.value = UiState.Success(data)
        }
    }
}
```

---

## まとめ

| 手法 | スコープ |
|------|---------|
| ErrorBoundary | Composable単位 |
| AsyncContent | 非同期状態管理 |
| CoroutineExceptionHandler | ViewModel |
| UncaughtExceptionHandler | アプリ全体 |

- ErrorBoundaryでComposableクラッシュを局所化
- AsyncContentでLoading/Success/Errorを統一管理
- CoroutineExceptionHandlerでViewModel内エラーを捕捉
- Crashlyticsで本番エラーを自動報告

---

8種類のAndroidアプリテンプレート（エラーハンドリング設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [エラーUI](https://zenn.dev/myougatheaxo/articles/android-compose-error-handling-ui-2026)
- [Result/Eitherパターン](https://zenn.dev/myougatheaxo/articles/android-compose-result-api-pattern-2026)
- [Crashlytics](https://zenn.dev/myougatheaxo/articles/android-compose-crash-analytics-2026)
