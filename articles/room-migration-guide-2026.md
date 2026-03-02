---
title: "Room Databaseマイグレーション入門 — テーブル変更でデータを失わない方法"
emoji: "🔄"
type: "tech"
topics: ["android", "kotlin", "room", "database"]
published: true
---

## この記事で学べること

アプリのバージョンアップでデータベースの構造を変更したい。でもユーザーの既存データは消したくない。**Room Migration**でデータを保持したままテーブル構造を変更する方法を解説します。

---

## マイグレーションが必要なとき

- テーブルにカラムを追加したい
- カラムの型を変更したい
- 新しいテーブルを追加したい
- インデックスを追加したい

**バージョン番号を上げる**ことで、Roomがマイグレーションを実行します。

---

## 基本パターン：カラム追加

### Before (version 1)

```kotlin
@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val createdAt: Long
)

@Database(entities = [Habit::class], version = 1)
abstract class AppDatabase : RoomDatabase()
```

### After (version 2) — `completed`カラムを追加

```kotlin
@Entity(tableName = "habits")
data class Habit(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val createdAt: Long,
    val completed: Boolean = false  // 新カラム
)

@Database(entities = [Habit::class], version = 2)
abstract class AppDatabase : RoomDatabase()
```

### Migration定義

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE habits ADD COLUMN completed INTEGER NOT NULL DEFAULT 0")
    }
}
```

### Database Builderに追加

```kotlin
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

---

## 複数バージョンのマイグレーション

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE habits ADD COLUMN completed INTEGER NOT NULL DEFAULT 0")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE habits ADD COLUMN category TEXT NOT NULL DEFAULT ''")
    }
}

// Builderに全て追加
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

version 1 → 3に一気にアップデートする場合、Roomは1→2→3と順番に実行してくれます。

---

## Auto Migration（Room 2.4+）

シンプルなカラム追加なら、手動でSQLを書かなくてもOK。

```kotlin
@Database(
    entities = [Habit::class],
    version = 3,
    autoMigrations = [
        AutoMigration(from = 1, to = 2),
        AutoMigration(from = 2, to = 3)
    ]
)
abstract class AppDatabase : RoomDatabase()
```

ただし、カラム名変更やテーブル削除には`@RenameColumn`や`@DeleteColumn`アノテーションが必要。

---

## 破壊的マイグレーション（最終手段）

```kotlin
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .fallbackToDestructiveMigration()
    .build()
```

**全データが消えます**。開発中のみ使用。本番では絶対に使わない。

---

## マイグレーションのテスト

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
        // version 1のDBを作成
        helper.createDatabase("test.db", 1).apply {
            execSQL("INSERT INTO habits (name, createdAt) VALUES ('Test', 0)")
            close()
        }

        // version 2にマイグレーション
        val db = helper.runMigrationsAndValidate("test.db", 2, true, MIGRATION_1_2)

        // データが保持されていることを確認
        val cursor = db.query("SELECT * FROM habits")
        assert(cursor.moveToFirst())
        assert(cursor.getString(cursor.getColumnIndex("name")) == "Test")
    }
}
```

---

## まとめ

- バージョン番号を上げてMigrationクラスを定義
- `ALTER TABLE`でカラム追加が最も一般的
- Auto Migration（Room 2.4+）でSQL不要な場合も
- `fallbackToDestructiveMigration`は開発中のみ
- マイグレーションは必ずテストする

---

8種類のAndroidアプリテンプレート（Room Database採用、マイグレーション対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Databaseでプライバシー設計入門](https://zenn.dev/myougatheaxo/articles/room-database-privacy-2026)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [DataStore移行ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-migration-2026)
