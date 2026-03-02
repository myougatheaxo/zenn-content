---
title: "ViewModel + StateFlow入門 — AIが書くAndroidアプリの設計パターン"
emoji: "🔄"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "mvvm", "stateflow"]
published: true
---

## この記事で学べること

AIでAndroidアプリを作ると、必ず`ViewModel`と`StateFlow`が登場します。「なぜこの構造なのか」「何をしているのか」をざっくり理解すれば、カスタマイズが格段に楽になります。

---

## ViewModelとは何か

**画面回転してもデータが消えない箱**です。

通常のActivityやComposableは、画面回転やバックグラウンド移行で破棄されます。ViewModelはその間もメモリに残り続けるため、データを保持できます。

```kotlin
class HabitViewModel(
    private val dao: HabitDao
) : ViewModel() {

    private val _habits = MutableStateFlow<List<Habit>>(emptyList())
    val habits: StateFlow<List<Habit>> = _habits.asStateFlow()

    init {
        viewModelScope.launch {
            dao.getAllHabits().collect { list ->
                _habits.value = list
            }
        }
    }
}
```

### 3つのポイント

1. **`MutableStateFlow`**: ViewModel内部で値を変更するための型
2. **`StateFlow`（公開用）**: 外部からは読み取り専用
3. **`viewModelScope.launch`**: ViewModelが破棄されたら自動キャンセル

---

## StateFlowとは何か

**値が変わったら自動で画面が更新される仕組み**です。

LiveDataの後継で、Kotlin Coroutineベース。Composeとの相性が最高です。

```kotlin
// ViewModel側
private val _count = MutableStateFlow(0)
val count: StateFlow<Int> = _count.asStateFlow()

fun increment() {
    _count.value += 1
}
```

```kotlin
// Composable側
@Composable
fun CounterScreen(viewModel: CounterViewModel) {
    val count by viewModel.count.collectAsState()

    Column {
        Text("Count: $count", style = MaterialTheme.typography.headlineLarge)
        Button(onClick = { viewModel.increment() }) {
            Text("＋1")
        }
    }
}
```

`collectAsState()`で購読するだけ。値が変わると自動でrecomposition（再描画）が走ります。

---

## UIState パターン

複数のStateFlowを1つにまとめるパターンです。AIが生成するコードでよく使われます。

```kotlin
data class HomeUiState(
    val habits: List<Habit> = emptyList(),
    val isLoading: Boolean = true,
    val error: String? = null
)

class HomeViewModel(private val dao: HabitDao) : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            try {
                dao.getAllHabits().collect { list ->
                    _uiState.value = _uiState.value.copy(
                        habits = list,
                        isLoading = false
                    )
                }
            } catch (e: Exception) {
                _uiState.value = _uiState.value.copy(
                    error = e.message,
                    isLoading = false
                )
            }
        }
    }
}
```

### メリット

- 画面の状態が1箇所に集約される
- `isLoading`、`error`を統一的に管理できる
- Composable側が`when`で分岐しやすい

```kotlin
@Composable
fun HomeScreen(viewModel: HomeViewModel) {
    val state by viewModel.uiState.collectAsState()

    when {
        state.isLoading -> CircularProgressIndicator()
        state.error != null -> Text("Error: ${state.error}")
        else -> LazyColumn {
            items(state.habits) { habit ->
                HabitItem(habit)
            }
        }
    }
}
```

---

## AIが生成するViewModelの特徴

Claude Codeで生成した8つのアプリのViewModelを分析：

| パターン | 採用率 |
|---------|-------|
| StateFlow（MutableStateFlow + asStateFlow） | 8/8 |
| UiState data class | 6/8 |
| viewModelScope.launch | 8/8 |
| Repository層あり | 3/8 |
| Room DAO直接参照 | 5/8 |

シンプルなアプリではDAO直接参照、複雑なアプリではRepository層を介す。AIは規模に応じて適切に判断しています。

---

## よくあるカスタマイズ

### 新しいアクションを追加する

```kotlin
// ViewModelに関数を追加
fun deleteHabit(habit: Habit) {
    viewModelScope.launch {
        dao.delete(habit)
    }
}
```

### フィルタリングを追加する

```kotlin
private val _searchQuery = MutableStateFlow("")

val filteredHabits = combine(_habits, _searchQuery) { habits, query ->
    if (query.isBlank()) habits
    else habits.filter { it.name.contains(query, ignoreCase = true) }
}.stateIn(viewModelScope, SharingStarted.Lazily, emptyList())
```

---

## まとめ

- **ViewModel** = 画面回転でも消えないデータ保持箱
- **StateFlow** = 値が変わったら画面が自動更新される仕組み
- **UiState** = 複数の状態を1つのdata classにまとめるパターン
- AIはこれらを自動で正しく実装してくれる

---

8種類のAndroidアプリテンプレート（全てViewModel + StateFlow + MVVM設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [Kotlin Coroutine入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-android-basics)
- [Room Databaseでプライバシー設計入門](https://zenn.dev/myougatheaxo/articles/room-database-privacy-2026)
- [CLAUDE.mdの書き方完全ガイド](https://zenn.dev/myougatheaxo/articles/claude-md-best-practices-2026)
