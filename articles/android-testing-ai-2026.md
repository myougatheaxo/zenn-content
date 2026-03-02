---
title: "AIが生成したAndroidアプリのテスト戦略 — 何をテストすべきか"
emoji: "🧪"
type: "tech"
topics: ["android", "kotlin", "testing", "jetpackcompose"]
published: true
---

## この記事で学べること

AIでアプリを生成すると、動くものはすぐ手に入ります。でも「本当に正しく動いているか」を確認するにはテストが必要です。

この記事では、**AIが生成したアプリに対して、最小限のテストで最大の安心を得る方法**を解説します。

---

## テストの優先順位

全部テストする必要はありません。コスパの良い順に並べると：

| 優先度 | テスト対象 | 理由 |
|--------|-----------|------|
| 1 | ViewModel | ビジネスロジックの中心 |
| 2 | Room DAO | データの永続化は壊れると致命的 |
| 3 | UI（Compose） | 表示の確認は手動でも可能 |

---

## ViewModelのテスト

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class HabitViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var viewModel: HabitViewModel
    private lateinit var fakeDao: FakeHabitDao

    @Before
    fun setup() {
        fakeDao = FakeHabitDao()
        viewModel = HabitViewModel(fakeDao)
    }

    @Test
    fun `addHabit increases list size`() = runTest {
        viewModel.addHabit("Morning Run")

        val habits = viewModel.habits.first()
        assertEquals(1, habits.size)
        assertEquals("Morning Run", habits[0].name)
    }

    @Test
    fun `deleteHabit removes from list`() = runTest {
        viewModel.addHabit("Morning Run")
        val habit = viewModel.habits.first()[0]

        viewModel.deleteHabit(habit)

        val habits = viewModel.habits.first()
        assertEquals(0, habits.size)
    }
}
```

### MainDispatcherRule

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
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
```

### FakeDao

```kotlin
class FakeHabitDao : HabitDao {
    private val habits = MutableStateFlow<List<Habit>>(emptyList())

    override fun getAllHabits(): Flow<List<Habit>> = habits

    override suspend fun insert(habit: Habit) {
        habits.value = habits.value + habit
    }

    override suspend fun delete(habit: Habit) {
        habits.value = habits.value - habit
    }
}
```

**ポイント**: 本物のデータベースは使わない。FakeのDAOを作って差し替える。

---

## Room DAOのテスト

```kotlin
@RunWith(AndroidJUnit4::class)
class HabitDaoTest {

    private lateinit var database: AppDatabase
    private lateinit var dao: HabitDao

    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        dao = database.habitDao()
    }

    @After
    fun teardown() {
        database.close()
    }

    @Test
    fun insertAndRetrieve() = runTest {
        val habit = Habit(name = "Test", createdAt = System.currentTimeMillis())
        dao.insert(habit)

        val result = dao.getAllHabits().first()
        assertEquals(1, result.size)
        assertEquals("Test", result[0].name)
    }
}
```

`inMemoryDatabaseBuilder`でテスト用のDBをメモリ上に作成。テストのたびにリセットされるため、副作用なし。

---

## Composeのテスト（必要な場合のみ）

```kotlin
@RunWith(AndroidJUnit4::class)
class HomeScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun displayHabitName() {
        composeTestRule.setContent {
            HabitItem(
                habit = Habit(name = "Exercise", createdAt = 0L),
                onDelete = {}
            )
        }

        composeTestRule
            .onNodeWithText("Exercise")
            .assertIsDisplayed()
    }
}
```

UIテストは書くコストが高い割に、手動確認でも代替可能。重要なUIだけテストすれば十分。

---

## テスト依存関係

```kotlin
// build.gradle.kts
dependencies {
    // Unit Test
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.0")

    // Android Test
    androidTestImplementation("androidx.test.ext:junit:1.2.1")
    androidTestImplementation("androidx.test:runner:1.6.1")
    androidTestImplementation("androidx.room:room-testing:2.6.1")
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
}
```

---

## AIに追加テストを書かせる

Claude Codeに「このViewModelのテストを書いて」と指示するだけで、上記のようなテストが自動生成されます。

```
claude "HabitViewModelのユニットテストを書いて。FakeDao使って"
```

テスト生成はAIが最も得意とする作業の1つ。テストパターンは定型的なので、AIの精度が高いです。

---

## まとめ

- **ViewModelテスト**が最優先（FakeDao + runTest）
- **Room DAOテスト**はinMemoryDatabaseBuilderで
- **UIテスト**は重要な画面だけ
- AIにテストを書かせるのが最も効率的

---

8種類のAndroidアプリテンプレート（テスト追加も簡単な構造）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Room Databaseでプライバシー設計入門](https://zenn.dev/myougatheaxo/articles/room-database-privacy-2026)
- [AIコードレビュー — ベテランエンジニアの評価](https://zenn.dev/myougatheaxo/articles/ai-code-quality-review)
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
