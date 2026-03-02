---
title: "Room Testing完全ガイド — InMemoryDB/DAOテスト/Migration検証"
emoji: "🧪"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Testing**（InMemoryDatabase、DAOユニットテスト、Migrationテスト）を解説します。

---

## InMemoryDatabase

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {
    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao

    @Before
    fun setup() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()
        userDao = database.userDao()
    }

    @After
    fun teardown() { database.close() }

    @Test
    fun insertAndGetUser() = runTest {
        val user = User(id = 1, name = "太郎", email = "taro@example.com")
        userDao.insert(user)

        val result = userDao.getById(1).first()
        assertThat(result).isEqualTo(user)
    }

    @Test
    fun deleteUser() = runTest {
        val user = User(id = 1, name = "太郎", email = "taro@example.com")
        userDao.insert(user)
        userDao.delete(user)

        val result = userDao.getById(1).first()
        assertThat(result).isNull()
    }
}
```

---

## Flowテスト

```kotlin
@Test
fun observeUsersFlow() = runTest {
    val users = userDao.getAll()

    users.test {
        // 初期状態
        assertThat(awaitItem()).isEmpty()

        // 追加
        userDao.insert(User(1, "太郎", "taro@example.com"))
        assertThat(awaitItem()).hasSize(1)

        // さらに追加
        userDao.insert(User(2, "花子", "hanako@example.com"))
        assertThat(awaitItem()).hasSize(2)

        cancelAndIgnoreRemainingEvents()
    }
}

@Test
fun searchUsers() = runTest {
    userDao.insert(User(1, "山田太郎", "taro@example.com"))
    userDao.insert(User(2, "田中花子", "hanako@example.com"))
    userDao.insert(User(3, "山田次郎", "jiro@example.com"))

    val results = userDao.searchByName("%山田%").first()
    assertThat(results).hasSize(2)
    assertThat(results.map { it.name }).containsExactly("山田太郎", "山田次郎")
}
```

---

## Migrationテスト

```kotlin
@RunWith(AndroidJUnit4::class)
class MigrationTest {
    @get:Rule
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java
    )

    @Test
    fun migrate1To2() {
        // v1でDB作成
        helper.createDatabase(TEST_DB, 1).apply {
            execSQL("INSERT INTO User (id, name) VALUES (1, '太郎')")
            close()
        }

        // v2にマイグレーション
        helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2)

        // v2で確認
        val db = helper.runMigrationsAndValidate(TEST_DB, 2, true)
        val cursor = db.query("SELECT * FROM User WHERE id = 1")
        cursor.moveToFirst()
        assertThat(cursor.getString(cursor.getColumnIndex("email"))).isNull()
    }

    companion object { const val TEST_DB = "migration-test" }
}
```

---

## まとめ

| パターン | 用途 |
|---------|------|
| `inMemoryDatabaseBuilder` | テスト用DB |
| `allowMainThreadQueries` | テスト簡略化 |
| `Turbine.test` | Flow検証 |
| `MigrationTestHelper` | Migration検証 |

- `inMemoryDatabaseBuilder`でテスト毎にクリーンなDB
- `allowMainThreadQueries`でテストスレッドから直接クエリ
- Turbineで`Flow`の値変更を順序検証
- `MigrationTestHelper`でスキーマ変更の安全確認

---

8種類のAndroidアプリテンプレート（テスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [Hilt Testing](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
