---
title: "コルーチンテスト完全ガイド — runTest/Turbine/TestDispatcher"
emoji: "🧪"
type: "tech"
topics: ["android", "kotlin", "coroutine", "testing"]
published: true
---

## この記事で学べること

**コルーチンテスト**（runTest、TestDispatcher、Turbine、Flow テスト、タイムアウト制御）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0")
    testImplementation("app.cash.turbine:turbine:1.2.0")
    testImplementation("io.mockk:mockk:1.13.13")
}
```

---

## runTest基本

```kotlin
@Test
fun fetchUserReturnsData() = runTest {
    val repository = UserRepository(FakeApiService())

    val user = repository.getUser("123")

    assertEquals("Alice", user.name)
}

// delay()は自動スキップ
@Test
fun delayIsSkipped() = runTest {
    val start = currentTime
    delay(10_000) // 実際には待たない
    assertEquals(10_000, currentTime - start)
}
```

---

## MainDispatcherRule

```kotlin
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

class UserViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun loadUserUpdatesState() = runTest {
        val viewModel = UserViewModel(FakeRepository())

        viewModel.loadUser("123")

        assertEquals("Alice", viewModel.uiState.value.user?.name)
    }
}
```

---

## TurbineでFlowテスト

```kotlin
@Test
fun observeUsersEmitsUpdates() = runTest {
    val repository = FakeUserRepository()

    repository.observeUsers().test {
        // 初期値
        assertEquals(emptyList(), awaitItem())

        // ユーザー追加
        repository.addUser(User("1", "Alice"))
        assertEquals(listOf(User("1", "Alice")), awaitItem())

        // さらに追加
        repository.addUser(User("2", "Bob"))
        val users = awaitItem()
        assertEquals(2, users.size)

        cancelAndIgnoreRemainingEvents()
    }
}

@Test
fun uiStateFlowTest() = runTest {
    val viewModel = TaskViewModel(FakeTaskRepository())

    viewModel.uiState.test {
        // 初期状態
        assertEquals(UiState.Loading, awaitItem())

        // データ読み込み完了
        val success = awaitItem()
        assertIs<UiState.Success>(success)
        assertEquals(3, success.tasks.size)

        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## StateFlow/SharedFlowテスト

```kotlin
@Test
fun stateFlowCollectsCorrectly() = runTest {
    val flow = MutableStateFlow(0)

    val values = mutableListOf<Int>()
    val job = launch(UnconfinedTestDispatcher(testScheduler)) {
        flow.collect { values.add(it) }
    }

    flow.value = 1
    flow.value = 2
    flow.value = 3

    assertEquals(listOf(0, 1, 2, 3), values)
    job.cancel()
}

@Test
fun sharedFlowEmitsEvents() = runTest {
    val events = MutableSharedFlow<String>()

    val received = mutableListOf<String>()
    val job = launch(UnconfinedTestDispatcher(testScheduler)) {
        events.collect { received.add(it) }
    }

    events.emit("event1")
    events.emit("event2")

    assertEquals(listOf("event1", "event2"), received)
    job.cancel()
}
```

---

## TestDispatcher使い分け

```kotlin
// UnconfinedTestDispatcher: 即座に実行（デフォルト推奨）
@Test
fun unconfinedTest() = runTest(UnconfinedTestDispatcher()) {
    val flow = flowOf(1, 2, 3)
    val result = flow.toList()
    assertEquals(listOf(1, 2, 3), result)
}

// StandardTestDispatcher: 明示的にadvance必要
@Test
fun standardTest() = runTest(StandardTestDispatcher()) {
    var result = 0
    launch { result = 42 }

    // まだ実行されていない
    assertEquals(0, result)

    advanceUntilIdle() // 実行を進める
    assertEquals(42, result)
}
```

---

## まとめ

| ツール | 用途 |
|--------|------|
| `runTest` | コルーチンテストスコープ |
| `UnconfinedTestDispatcher` | 即座実行 |
| `StandardTestDispatcher` | 制御付き実行 |
| `Turbine` | Flow値の順次検証 |
| `MainDispatcherRule` | Dispatchers.Main差し替え |

- `runTest`でdelay自動スキップ
- `MainDispatcherRule`でViewModel テスト可能
- `Turbine`の`test{}`でFlow値を順次assert
- `advanceUntilIdle()`で保留コルーチン実行

---

8種類のAndroidアプリテンプレート（テスト完備）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel単体テスト](https://zenn.dev/myougatheaxo/articles/android-compose-unit-test-viewmodel-2026)
- [コルーチン基礎](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-basics-2026)
- [Flow/Channel](https://zenn.dev/myougatheaxo/articles/kotlin-flow-channel-2026)
