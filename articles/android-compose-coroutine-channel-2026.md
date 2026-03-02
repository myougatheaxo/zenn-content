---
title: "Coroutine Channel完全ガイド — Channel/SharedFlow/EventBus/単発イベント"
emoji: "📡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Coroutine Channel**（Channel、SharedFlow、単発イベント配信、EventBusパターン）を解説します。

---

## Channel（単発イベント）

```kotlin
class ItemViewModel @Inject constructor(
    private val repository: ItemRepository
) : ViewModel() {
    private val _event = Channel<UiEvent>(Channel.BUFFERED)
    val event = _event.receiveAsFlow()

    fun deleteItem(id: Int) {
        viewModelScope.launch {
            try {
                repository.delete(id)
                _event.send(UiEvent.ShowSnackbar("削除しました", "元に戻す"))
            } catch (e: Exception) {
                _event.send(UiEvent.ShowSnackbar("削除に失敗しました"))
            }
        }
    }

    fun navigateToDetail(id: Int) {
        viewModelScope.launch {
            _event.send(UiEvent.Navigate("detail/$id"))
        }
    }
}

sealed class UiEvent {
    data class ShowSnackbar(val message: String, val action: String? = null) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
    data object NavigateBack : UiEvent()
}
```

---

## Composeでイベント受信

```kotlin
@Composable
fun ItemScreen(viewModel: ItemViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }
    val navController = LocalNavController.current

    LaunchedEffect(Unit) {
        viewModel.event.collect { event ->
            when (event) {
                is UiEvent.ShowSnackbar -> {
                    val result = snackbarHostState.showSnackbar(
                        message = event.message,
                        actionLabel = event.action
                    )
                    if (result == SnackbarResult.ActionPerformed) {
                        viewModel.undoDelete()
                    }
                }
                is UiEvent.Navigate -> navController.navigate(event.route)
                is UiEvent.NavigateBack -> navController.popBackStack()
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        // コンテンツ
    }
}
```

---

## SharedFlow（マルチサブスクライバ）

```kotlin
class AppEventBus @Inject constructor() {
    private val _events = MutableSharedFlow<AppEvent>(
        replay = 0,
        extraBufferCapacity = 10,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events = _events.asSharedFlow()

    suspend fun emit(event: AppEvent) {
        _events.emit(event)
    }
}

sealed class AppEvent {
    data class UserLoggedIn(val userId: String) : AppEvent()
    data object UserLoggedOut : AppEvent()
    data class ThemeChanged(val isDark: Boolean) : AppEvent()
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Channel` | 単発イベント（1:1） |
| `SharedFlow` | マルチキャスト（1:N） |
| `StateFlow` | 状態保持 |
| `receiveAsFlow` | Channel→Flow変換 |

- `Channel`でSnackbar/ナビゲーションの単発イベント
- `SharedFlow`で複数購読者へのイベント配信
- `receiveAsFlow`でComposeのLaunchedEffectと連携
- `StateFlow`は状態、`Channel/SharedFlow`はイベントに使い分け

---

8種類のAndroidアプリテンプレート（イベント設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [副作用API](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effects-2026)
- [Coroutine Testing](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-testing-2026)
