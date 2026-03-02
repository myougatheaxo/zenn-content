---
title: "Room AutoMigration完全ガイド — 自動マイグレーション/AutoMigrationSpec/スキーマ管理"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room AutoMigration**（自動マイグレーション、AutoMigrationSpec、スキーマエクスポート）を解説します。

---

## AutoMigration基本

```kotlin
@Database(
    entities = [User::class, Note::class],
    version = 3,
    autoMigrations = [
        AutoMigration(from = 1, to = 2),
        AutoMigration(from = 2, to = 3, spec = Migration2To3::class)
    ],
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun noteDao(): NoteDao
}
```

---

## AutoMigrationSpec

```kotlin
// カラム名変更時
@RenameColumn(tableName = "User", fromColumnName = "userName", toColumnName = "name")
class Migration2To3 : AutoMigrationSpec

// テーブル名変更時
@RenameTable(fromTableName = "OldNote", toTableName = "Note")
class Migration3To4 : AutoMigrationSpec

// カラム削除時
@DeleteColumn(tableName = "User", columnName = "tempField")
class Migration4To5 : AutoMigrationSpec

// 複合変更
@RenameColumn(tableName = "Note", fromColumnName = "text", toColumnName = "content")
@DeleteColumn(tableName = "Note", columnName = "deprecated_field")
class Migration5To6 : AutoMigrationSpec
```

---

## スキーマエクスポート設定

```kotlin
// build.gradle.kts
android {
    defaultConfig {
        ksp {
            arg("room.schemaLocation", "$projectDir/schemas")
        }
    }
}

// テスト用にスキーマを含める
android {
    sourceSets {
        getByName("androidTest").assets.srcDirs("$projectDir/schemas")
    }
}
```

---

## 手動+自動の組み合わせ

```kotlin
@Database(
    entities = [User::class],
    version = 4,
    autoMigrations = [
        AutoMigration(from = 1, to = 2),  // カラム追加のみ
        AutoMigration(from = 3, to = 4, spec = Spec3To4::class)  // リネーム
    ]
)
abstract class AppDatabase : RoomDatabase()

// v2→v3は複雑なためマニュアル
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("CREATE TABLE User_new (id INTEGER PRIMARY KEY, name TEXT NOT NULL, email TEXT)")
        db.execSQL("INSERT INTO User_new SELECT id, name, null FROM User")
        db.execSQL("DROP TABLE User")
        db.execSQL("ALTER TABLE User_new RENAME TO User")
    }
}

// DB構築
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_2_3)
    .build()
```

---

## まとめ

| 機能 | 用途 |
|------|------|
| `autoMigrations` | 自動マイグレーション定義 |
| `AutoMigrationSpec` | 追加情報（リネーム等） |
| `@RenameColumn` | カラム名変更 |
| `@DeleteColumn` | カラム削除 |

- `AutoMigration`でカラム追加/削除を自動処理
- `AutoMigrationSpec`でリネーム等の補助情報提供
- `exportSchema = true`でスキーマJSON出力
- 複雑な変更は手動Migrationと組み合わせ

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Migration(手動)](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [Room Testing](https://zenn.dev/myougatheaxo/articles/android-compose-room-testing-2026)
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
