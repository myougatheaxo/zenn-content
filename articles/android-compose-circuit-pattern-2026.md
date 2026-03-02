---
title: "Circuit/Molecule パターン完全ガイド — Presenter分離/状態管理"
emoji: "🔌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "architecture"]
published: true
---

## この記事で学べること

**Circuit/Moleculeパターン**（Presenter分離、UIイベント駆動、状態マシン、テスタビリティ向上）を解説します。

---

## Circuit パターンとは

Slackが開発したアーキテクチャ。Screen/Presenter/UIの明確な分離でテスタビリティを最大化します。

---

## 基本構成

```kotlin
// Screen: 画面の定義
@Parcelize
data object HomeScreen : Screen {
    data class State(
        val items: List<Item> = emptyList(),
        val isLoading: Boolean = false,
        val eventSink: (Event) -> Unit = {}
    ) : CircuitUiState

    sealed interface Event : CircuitUiEvent {
        data object Refresh : Event
        data class ItemClick(val itemId: String) : Event
        data class Delete(val itemId: String) : Event
    }
}
```

---

## Presenter

```kotlin
class HomePresenter @Inject constructor(
    private val repository: ItemRepository,
    private val navigator: Navigator
) : Presenter<HomeScreen.State> {

    @Composable
    override fun present(): HomeScreen.State {
        var items by remember { mutableStateOf<List<Item>>(emptyList()) }
        var isLoading by remember { mutableStateOf(true) }

        LaunchedEffect(Unit) {
            repository.getItems().collect {
                items = it
                isLoading = false
            }
        }

        return HomeScreen.State(
            items = items,
            isLoading = isLoading
        ) { event ->
            when (event) {
                HomeScreen.Event.Refresh -> {
                    isLoading = true
                    // Refresh logic
                }
                is HomeScreen.Event.ItemClick -> {
                    navigator.goTo(DetailScreen(event.itemId))
                }
                is HomeScreen.Event.Delete -> {
                    items = items.filter { it.id != event.itemId }
                }
            }
        }
    }
}
```

---

## UI

```kotlin
@CircuitInject(HomeScreen::class, AppScope::class)
@Composable
fun HomeUi(state: HomeScreen.State, modifier: Modifier = Modifier) {
    Scaffold(modifier) { padding ->
        if (state.isLoading) {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        } else {
            LazyColumn(contentPadding = padding) {
                items(state.items, key = { it.id }) { item ->
                    ListItem(
                        headlineContent = { Text(item.title) },
                        modifier = Modifier.clickable {
                            state.eventSink(HomeScreen.Event.ItemClick(item.id))
                        },
                        trailingContent = {
                            IconButton(onClick = {
                                state.eventSink(HomeScreen.Event.Delete(item.id))
                            }) {
                                Icon(Icons.Default.Delete, "削除")
                            }
                        }
                    )
                }
            }
        }
    }
}
```

---

## Moleculeパターン（簡易版）

```kotlin
// Molecule: Composable関数でStateを生成
@Composable
fun counterPresenter(events: Flow<CounterEvent>): CounterState {
    var count by remember { mutableIntStateOf(0) }

    LaunchedEffect(Unit) {
        events.collect { event ->
            when (event) {
                CounterEvent.Increment -> count++
                CounterEvent.Decrement -> count--
                CounterEvent.Reset -> count = 0
            }
        }
    }

    return CounterState(count = count)
}

data class CounterState(val count: Int)
sealed interface CounterEvent {
    data object Increment : CounterEvent
    data object Decrement : CounterEvent
    data object Reset : CounterEvent
}
```

---

## テスト

```kotlin
@Test
fun presenterEmitsCorrectState() = runTest {
    val fakeRepository = FakeItemRepository(listOf(testItem1, testItem2))
    val fakeNavigator = FakeNavigator()
    val presenter = HomePresenter(fakeRepository, fakeNavigator)

    moleculeFlow(RecompositionMode.Immediate) { presenter.present() }.test {
        val initialState = awaitItem()
        assertTrue(initialState.isLoading)

        val loadedState = awaitItem()
        assertFalse(loadedState.isLoading)
        assertEquals(2, loadedState.items.size)

        loadedState.eventSink(HomeScreen.Event.Delete(testItem1.id))
        val afterDelete = awaitItem()
        assertEquals(1, afterDelete.items.size)
    }
}
```

---

## まとめ

| 要素 | 役割 |
|------|------|
| Screen | 画面定義 + State + Event |
| Presenter | 状態管理ロジック |
| UI | 純粋な描画関数 |
| eventSink | UI→Presenterイベント |

- Screen/Presenter/UIの明確な責務分離
- UIはState受け取りのみ（テストで簡単に差し替え）
- eventSinkパターンでイベントを統一管理
- Moleculeでテスタブルなpresenter関数

---

8種類のAndroidアプリテンプレート（モダンアーキテクチャ採用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [クリーンアーキテクチャ](https://zenn.dev/myougatheaxo/articles/android-compose-clean-architecture-2026)
- [ViewModel/Hilt](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-hilt-2026)
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
