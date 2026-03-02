---
title: "Flow + Lifecycleガイド — collectAsStateWithLifecycle完全解説"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "flow"]
published: true
---

## この記事で学べること

Composeでの**Flow収集とライフサイクル管理**（collectAsStateWithLifecycle、repeatOnLifecycle）を解説します。

---

## collectAsStateWithLifecycle

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = viewModel()) {
    // ライフサイクル対応のFlow収集（推奨）
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // バックグラウンドでは収集を自動停止
    // → バッテリー消費を抑制

    when (val state = uiState) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> UserContent(state.data)
        is UiState.Error -> ErrorView(state.message)
    }
}
```

---

## collectAsState vs collectAsStateWithLifecycle

```kotlin
// ❌ collectAsState: バックグラウンドでも収集し続ける
val state by viewModel.flow.collectAsState(initial = emptyList())

// ✅ collectAsStateWithLifecycle: STARTED以上で収集
val state by viewModel.flow.collectAsStateWithLifecycle(initialValue = emptyList())

// ライフサイクル状態の指定
val state by viewModel.flow.collectAsStateWithLifecycle(
    initialValue = emptyList(),
    minActiveState = Lifecycle.State.RESUMED // RESUMEDの時のみ収集
)
```

---

## ViewModelでのFlow設計

```kotlin
class NewsViewModel(private val repository: NewsRepository) : ViewModel() {

    // 1. stateIn: Flow → StateFlow変換
    val news: StateFlow<List<News>> = repository.getNewsFlow()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000), // 5秒後に停止
            initialValue = emptyList()
        )

    // 2. 複数Flowの結合
    private val _searchQuery = MutableStateFlow("")
    private val _category = MutableStateFlow("all")

    val filteredNews: StateFlow<List<News>> = combine(
        _searchQuery, _category, repository.getNewsFlow()
    ) { query, category, news ->
        news.filter { item ->
            (query.isBlank() || item.title.contains(query, ignoreCase = true)) &&
            (category == "all" || item.category == category)
        }
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun updateQuery(query: String) { _searchQuery.value = query }
    fun updateCategory(category: String) { _category.value = category }
}
```

---

## SharingStartedの使い分け

```kotlin
// Eagerly: 即座に開始、ViewModelが破棄されるまで継続
.stateIn(viewModelScope, SharingStarted.Eagerly, initialValue)

// Lazily: 最初のサブスクライバーで開始、以後継続
.stateIn(viewModelScope, SharingStarted.Lazily, initialValue)

// WhileSubscribed: サブスクライバーがいる間のみ
// stopTimeoutMillis: 最後のサブスクライバーが消えてから停止するまでの時間
// replayExpirationMillis: 停止後にキャッシュを保持する時間
.stateIn(
    viewModelScope,
    SharingStarted.WhileSubscribed(
        stopTimeoutMillis = 5000,
        replayExpirationMillis = Long.MAX_VALUE
    ),
    initialValue
)
```

---

## SharedFlowでイベント処理

```kotlin
class FormViewModel : ViewModel() {

    // 一度きりのイベント（Snackbar、画面遷移など）
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun submit() {
        viewModelScope.launch {
            try {
                repository.save()
                _events.emit(UiEvent.NavigateBack)
            } catch (e: Exception) {
                _events.emit(UiEvent.ShowSnackbar(e.message ?: "エラー"))
            }
        }
    }
}

sealed interface UiEvent {
    data object NavigateBack : UiEvent
    data class ShowSnackbar(val message: String) : UiEvent
}

@Composable
fun FormScreen(viewModel: FormViewModel = viewModel(), onBack: () -> Unit) {
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.NavigateBack -> onBack()
                is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
            }
        }
    }

    // UI...
}
```

---

## まとめ

- `collectAsStateWithLifecycle()`が推奨（バッテリー効率）
- `SharingStarted.WhileSubscribed(5000)`で画面回転対応
- `combine`で複数Flowの結合
- `SharedFlow`で一度きりのイベント配信
- `stateIn`でFlow → StateFlow変換
- `minActiveState`でライフサイクル制御

---

8種類のAndroidアプリテンプレート（Flow+Lifecycle設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutines/Flow完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
- [ViewModel/StateFlowガイド](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [状態管理ガイド](https://zenn.dev/myougatheaxo/articles/compose-state-management-2026)
