---
title: "Room DB マイグレーション完全ガイド — 自動/手動/テスト"
emoji: "🗃️"
type: "tech"
topics: ["android", "kotlin", "room", "database"]
published: true
---

## この記事で学べること

**Room**のデータベースマイグレーション（自動マイグレーション、手動Migration、テスト）を解説します。

---

## 自動マイグレーション (Room 2.4+)

```kotlin
@Database(
    entities = [User::class],
    version = 2,
    autoMigrations = [
        AutoMigration(from = 1, to = 2)
    ]
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// v1
@Entity
data class User(
    @PrimaryKey val id: String,
    val name: String
)

// v2: emailカラム追加
@Entity
data class User(
    @PrimaryKey val id: String,
    val name: String,
    @ColumnInfo(defaultValue = "")
    val email: String = ""
)
```

---

## カラム名変更/削除時のSpec

```kotlin
@RenameColumn(tableName = "User", fromColumnName = "user_name", toColumnName = "name")
class RenameSpec : AutoMigrationSpec

@DeleteColumn(tableName = "User", columnName = "old_field")
class DeleteSpec : AutoMigrationSpec

@Database(
    entities = [User::class],
    version = 3,
    autoMigrations = [
        AutoMigration(from = 1, to = 2),
        AutoMigration(from = 2, to = 3, spec = RenameSpec::class)
    ]
)
abstract class AppDatabase : RoomDatabase()
```

---

## 手動マイグレーション

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE User ADD COLUMN email TEXT NOT NULL DEFAULT ''")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // テーブル再作成（SQLiteではカラム削除不可のため）
        db.execSQL("""
            CREATE TABLE User_new (
                id TEXT NOT NULL PRIMARY KEY,
                name TEXT NOT NULL,
                email TEXT NOT NULL DEFAULT ''
            )
        """)
        db.execSQL("INSERT INTO User_new (id, name, email) SELECT id, name, email FROM User")
        db.execSQL("DROP TABLE User")
        db.execSQL("ALTER TABLE User_new RENAME TO User")
    }
}

// DB構築時に指定
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
        // v1のDBを作成
        helper.createDatabase("test-db", 1).apply {
            execSQL("INSERT INTO User (id, name) VALUES ('1', 'Test')")
            close()
        }

        // v2にマイグレーション
        val db = helper.runMigrationsAndValidate("test-db", 2, true, MIGRATION_1_2)

        // 検証
        val cursor = db.query("SELECT * FROM User WHERE id = '1'")
        cursor.moveToFirst()
        assertEquals("", cursor.getString(cursor.getColumnIndex("email")))
        cursor.close()
    }
}
```

---

## まとめ

- Room 2.4+の`autoMigrations`で簡単なスキーマ変更を自動処理
- `@RenameColumn`/`@DeleteColumn`で名前変更/削除に対応
- 複雑な変更は`Migration`クラスで手動SQL実行
- `fallbackToDestructiveMigration()`はデータ消失するので本番では避ける
- `MigrationTestHelper`でマイグレーションをテスト
- バージョン番号は必ずインクリメント

---

8種類のAndroidアプリテンプレート（Room DB設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Paging3 + Room](https://zenn.dev/myougatheaxo/articles/android-compose-paging-room-2026)
- [ファイルストレージ](https://zenn.dev/myougatheaxo/articles/android-file-storage-2026)
- [Hilt依存性注入](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
