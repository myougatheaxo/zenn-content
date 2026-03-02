---
title: "Compose Room Migration完全ガイド — スキーマ変更/自動マイグレーション/手動マイグレーション/テスト"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Compose Room Migration**（自動マイグレーション、手動マイグレーション、destructiveマイグレーション、テスト）を解説します。

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

// カラム名変更時のspec
@RenameColumn(tableName = "users", fromColumnName = "user_name", toColumnName = "display_name")
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
                completed INTEGER NOT NULL DEFAULT 0,
                user_id INTEGER NOT NULL,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
            )
        """)
        db.execSQL("CREATE INDEX index_tasks_user_id ON tasks(user_id)")
    }
}

// DI
@Provides
@Singleton
fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
    Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
        .build()
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
        // v1のDB作成
        helper.createDatabase("test.db", 1).apply {
            execSQL("INSERT INTO users (id, user_name) VALUES (1, 'テスト')")
            close()
        }

        // v2にマイグレーション
        val db = helper.runMigrationsAndValidate("test.db", 2, true, MIGRATION_1_2)

        // 検証
        val cursor = db.query("SELECT * FROM users WHERE id = 1")
        assertTrue(cursor.moveToFirst())
        assertEquals("", cursor.getString(cursor.getColumnIndex("email")))
        cursor.close()
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `AutoMigration` | 自動マイグレーション |
| `Migration` | 手動マイグレーション |
| `MigrationTestHelper` | テスト |
| `fallbackToDestructiveMigration` | 破壊的更新 |

- Room 2.4+の`autoMigrations`で簡単なスキーマ変更を自動化
- `@RenameColumn`/`@DeleteColumn`で自動マイグレーションをカスタマイズ
- 複雑な変更は`Migration`クラスで手動SQLを記述
- `MigrationTestHelper`で本番前にマイグレーションを検証

---

8種類のAndroidアプリテンプレート（Room対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Room](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-2026)
- [Compose Room Relation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-relation-2026)
- [Compose Room FTS](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-fts-2026)
