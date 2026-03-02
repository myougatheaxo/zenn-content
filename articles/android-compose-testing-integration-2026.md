---
title: "Compose統合テスト完全ガイド — Navigation/Hilt/Room"
emoji: "🔬"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

Composeの**統合テスト**（Navigation テスト、Hilt テスト、Room + UI テスト）を解説します。

---

## Navigation テスト

```kotlin
@HiltAndroidTest
class NavigationTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun navigateToDetail() {
        composeRule.apply {
            // ホーム画面の確認
            onNodeWithText("ホーム").assertIsDisplayed()

            // アイテムをタップ
            onNodeWithText("Item 1").performClick()

            // 詳細画面に遷移
            onNodeWithText("Item 1 の詳細").assertIsDisplayed()
        }
    }

    @Test
    fun navigateBack() {
        composeRule.apply {
            onNodeWithText("Item 1").performClick()
            onNodeWithText("Item 1 の詳細").assertIsDisplayed()

            // 戻るボタン
            onNodeWithContentDescription("戻る").performClick()
            onNodeWithText("ホーム").assertIsDisplayed()
        }
    }
}
```

---

## TestNavHost

```kotlin
@Composable
fun TestNavHost(
    navController: NavHostController = rememberNavController(),
    startDestination: String = "home"
) {
    NavHost(navController = navController, startDestination = startDestination) {
        composable("home") {
            HomeScreen(onNavigate = { navController.navigate("detail/$it") })
        }
        composable("detail/{id}") {
            DetailScreen()
        }
    }
}

@Test
fun testNavigationFlow() {
    val navController = TestNavHostController(ApplicationProvider.getApplicationContext())

    composeRule.setContent {
        navController.navigatorProvider.addNavigator(ComposeNavigator())
        TestNavHost(navController = navController)
    }

    // 遷移先を検証
    composeRule.onNodeWithText("Item 1").performClick()
    assertEquals("detail/1", navController.currentDestination?.route)
}
```

---

## Hilt統合テスト

```kotlin
@HiltAndroidTest
@UninstallModules(RepositoryModule::class)
class UserScreenTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<TestActivity>()

    @Module
    @InstallIn(SingletonComponent::class)
    object TestModule {
        @Provides
        @Singleton
        fun provideUserRepository(): UserRepository = FakeUserRepository()
    }

    @Before
    fun setup() {
        hiltRule.inject()
    }

    @Test
    fun showsUserList() {
        composeRule.setContent {
            UserListScreen()
        }

        composeRule.onNodeWithText("Alice").assertIsDisplayed()
        composeRule.onNodeWithText("Bob").assertIsDisplayed()
    }

    @Test
    fun deleteUser() {
        composeRule.setContent {
            UserListScreen()
        }

        // スワイプ削除
        composeRule.onNodeWithText("Alice").performTouchInput {
            swipeLeft()
        }

        composeRule.onNodeWithText("Alice").assertDoesNotExist()
    }
}

class FakeUserRepository : UserRepository {
    private val users = mutableListOf(
        User("1", "Alice"),
        User("2", "Bob")
    )

    override fun observeUsers() = flowOf(users.toList())
    override suspend fun delete(id: String) {
        users.removeAll { it.id == id }
    }
}
```

---

## Room統合テスト

```kotlin
@HiltAndroidTest
class TaskDaoTest {

    @get:Rule
    val hiltRule = HiltAndroidRule(this)

    private lateinit var db: AppDatabase
    private lateinit var dao: TaskDao

    @Before
    fun setup() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        dao = db.taskDao()
    }

    @After
    fun teardown() {
        db.close()
    }

    @Test
    fun insertAndObserve() = runTest {
        val task = TaskEntity(title = "Test", description = "Desc")
        dao.insert(task)

        dao.observeAll().first().let { tasks ->
            assertEquals(1, tasks.size)
            assertEquals("Test", tasks[0].title)
        }
    }

    @Test
    fun toggleCompleted() = runTest {
        val id = dao.insert(TaskEntity(title = "Task"))
        dao.toggleCompleted(id)

        val task = dao.getById(id)
        assertTrue(task!!.isCompleted)
    }
}
```

---

## End-to-End テスト

```kotlin
@HiltAndroidTest
class AddTaskE2ETest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val composeRule = createAndroidComposeRule<MainActivity>()

    @Test
    fun addNewTask_showsInList() {
        composeRule.apply {
            // FABをタップ
            onNodeWithContentDescription("タスク追加").performClick()

            // タイトル入力
            onNodeWithTag("task_title_input").performTextInput("新しいタスク")

            // 保存
            onNodeWithText("保存").performClick()

            // リストに表示される
            onNodeWithText("新しいタスク").assertIsDisplayed()
        }
    }
}
```

---

## まとめ

| テスト種別 | ツール | 対象 |
|-----------|--------|------|
| UIテスト | `createComposeRule` | 単一Composable |
| Navigation | `TestNavHostController` | 画面遷移 |
| Hilt統合 | `@HiltAndroidTest` | DI+UI |
| Room | `inMemoryDatabaseBuilder` | DB操作 |
| E2E | `createAndroidComposeRule` | フルフロー |

- `@UninstallModules`でテスト用モジュール差し替え
- `FakeRepository`で安定したテストデータ
- `Room.inMemoryDatabaseBuilder`で高速DBテスト
- `performTextInput`/`performClick`でUI操作

---

8種類のAndroidアプリテンプレート（テスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ComposeTestRule](https://zenn.dev/myougatheaxo/articles/android-compose-testing-compose-rule-2026)
- [ViewModel単体テスト](https://zenn.dev/myougatheaxo/articles/android-compose-unit-test-viewmodel-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-dependency-injection-hilt-2026)
