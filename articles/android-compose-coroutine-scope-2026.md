---
title: "Coroutine Scope完全ガイド — viewModelScope/lifecycleScope/カスタムScope"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "coroutine"]
published: true
---

## この記事で学べること

**Coroutine Scope**（viewModelScope、lifecycleScope、rememberCoroutineScope、カスタムScope）を解説します。

---

## Android標準Scope

```kotlin
// viewModelScope: ViewModelのライフサイクルに紐づく
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repository: HomeRepository
) : ViewModel() {
    init {
        viewModelScope.launch {
            // ViewModelがクリアされると自動キャンセル
            repository.getItems().collect { items ->
                _uiState.value = UiState.Success(items)
            }
        }
    }
}

// lifecycleScope: Activity/Fragmentのライフサイクル
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            // Activityが破棄されると自動キャンセル
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.events.collect { handleEvent(it) }
            }
        }
    }
}
```

---

## Compose内のScope

```kotlin
@Composable
fun ScopeDemo() {
    // rememberCoroutineScope: Composableのライフサイクル
    val scope = rememberCoroutineScope()
    val snackbarHostState = remember { SnackbarHostState() }

    Button(onClick = {
        scope.launch {
            snackbarHostState.showSnackbar("保存しました")
        }
    }) { Text("保存") }

    // LaunchedEffect: キーが変わると再起動
    var userId by remember { mutableStateOf("user1") }
    LaunchedEffect(userId) {
        // userIdが変わると前のcoroutineはキャンセルされ再起動
        val profile = api.getProfile(userId)
        // ...
    }
}
```

---

## カスタムScope

```kotlin
// アプリ全体のScope（DIで提供）
@Module
@InstallIn(SingletonComponent::class)
object CoroutineScopeModule {
    @Provides @Singleton @ApplicationScope
    fun provideApplicationScope(): CoroutineScope =
        CoroutineScope(SupervisorJob() + Dispatchers.Default)
}

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class ApplicationScope

// 使用: アプリ全体で動くバックグラウンドタスク
class SyncManager @Inject constructor(
    @ApplicationScope private val scope: CoroutineScope,
    private val repository: DataRepository
) {
    fun startPeriodicSync() {
        scope.launch {
            while (isActive) {
                repository.sync()
                delay(15.minutes)
            }
        }
    }
}
```

---

## まとめ

| Scope | ライフサイクル |
|-------|---------------|
| `viewModelScope` | ViewModel |
| `lifecycleScope` | Activity/Fragment |
| `rememberCoroutineScope` | Composable |
| `LaunchedEffect` | キー依存 |

- `viewModelScope`はViewModel破棄時に自動キャンセル
- `lifecycleScope`+`repeatOnLifecycle`でライフサイクル安全にFlow収集
- `rememberCoroutineScope`はイベントハンドラ用
- `SupervisorJob()`で子の失敗が親に波及しない

---

8種類のAndroidアプリテンプレート（Coroutine対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Coroutine Dispatcher](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-dispatcher-2026)
- [Coroutine Cancellation](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-cancellation-2026)
- [Coroutine ExceptionHandler](https://zenn.dev/myougatheaxo/articles/android-compose-coroutine-exception-handler-2026)
