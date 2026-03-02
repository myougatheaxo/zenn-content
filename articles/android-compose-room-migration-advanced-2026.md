---
title: "Room Migration完全ガイド — 自動/手動マイグレーション/テスト/フォールバック"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Migration**（自動マイグレーション、手動マイグレーション、テスト、破壊的フォールバック）を解説します。

---

## 自動マイグレーション

```kotlin
@Database(
    entities = [User::class, Task::class],
    version = 3,
    autoMigrations = [
        AutoMigration(from = 1, to = 2),
        AutoMigration(from = 2, to = 3, spec = Migration2To3::class)
    ]
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun taskDao(): TaskDao
}

@RenameColumn(tableName = "users", fromColumnName = "name", toColumnName = "full_name")
class Migration2To3 : AutoMigrationSpec
```

---

## 手動マイグレーション

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN email TEXT NOT NULL DEFAULT ''")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // 新テーブル作成
        db.execSQL("""
            CREATE TABLE tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
                title TEXT NOT NULL,
                user_id INTEGER NOT NULL,
                completed INTEGER NOT NULL DEFAULT 0,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
            )
        """)
        db.execSQL("CREATE INDEX index_tasks_user_id ON tasks(user_id)")
    }
}

// Database構築
@Provides
@Singleton
fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
    return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
        .build()
}
```

---

## マイグレーションテスト

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
        // Version 1 のDB作成
        helper.createDatabase("test.db", 1).apply {
            execSQL("INSERT INTO users (id, name) VALUES (1, 'テスト')")
            close()
        }

        // Version 2 へマイグレーション
        val db = helper.runMigrationsAndValidate("test.db", 2, true, MIGRATION_1_2)

        val cursor = db.query("SELECT * FROM users WHERE id = 1")
        assertTrue(cursor.moveToFirst())
        assertEquals("", cursor.getString(cursor.getColumnIndex("email")))
        cursor.close()
    }
}
```

---

## 破壊的フォールバック

```kotlin
@Provides
@Singleton
fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
    return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
        .fallbackToDestructiveMigration()  // マイグレーション未定義時はDB再作成
        .build()
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| 自動 | カラム追加/リネーム |
| 手動 | テーブル構造変更 |
| テスト | `MigrationTestHelper` |
| フォールバック | 開発中のみ |

- `autoMigrations`でシンプルな変更は自動対応
- 手動マイグレーションでSQL直接実行
- `MigrationTestHelper`でマイグレーションテスト
- 本番では`fallbackToDestructiveMigration`を避ける

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [TypeConverter](https://zenn.dev/myougatheaxo/articles/android-compose-room-typeconverter-2026)
- [全文検索](https://zenn.dev/myougatheaxo/articles/android-compose-fts-search-2026)
