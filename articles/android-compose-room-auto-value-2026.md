---
title: "Room AutoGenerate/Default完全ガイド — 自動生成ID/デフォルト値/TypeConverter"
emoji: "🔢"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room自動生成・デフォルト値**（autoGenerate、defaultValue、TypeConverter、Enum変換）を解説します。

---

## autoGenerate

```kotlin
@Entity
data class Note(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val title: String,
    val content: String,
    @ColumnInfo(defaultValue = "CURRENT_TIMESTAMP")
    val createdAt: String? = null,
    @ColumnInfo(defaultValue = "0")
    val isPinned: Boolean = false
)

@Dao
interface NoteDao {
    @Insert
    suspend fun insert(note: Note): Long  // 生成されたIDを返す

    @Insert
    suspend fun insertAll(notes: List<Note>): List<Long>

    @Query("SELECT * FROM Note ORDER BY isPinned DESC, createdAt DESC")
    fun getAll(): Flow<List<Note>>
}
```

---

## TypeConverter

```kotlin
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? = date?.time

    @TypeConverter
    fun fromStringList(value: String?): List<String> =
        value?.split(",")?.filter { it.isNotBlank() } ?: emptyList()

    @TypeConverter
    fun stringListToString(list: List<String>): String =
        list.joinToString(",")

    @TypeConverter
    fun fromPriority(priority: Priority): String = priority.name

    @TypeConverter
    fun toPriority(value: String): Priority = Priority.valueOf(value)
}

enum class Priority { LOW, MEDIUM, HIGH, URGENT }

@Database(entities = [Note::class, Task::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
    abstract fun taskDao(): TaskDao
}
```

---

## Enum + defaultValue

```kotlin
@Entity
data class Task(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    val title: String,
    val priority: Priority = Priority.MEDIUM,
    @ColumnInfo(defaultValue = "0")
    val isCompleted: Boolean = false,
    val tags: List<String> = emptyList(),
    val dueDate: Date? = null
)

@Dao
interface TaskDao {
    @Query("SELECT * FROM Task WHERE isCompleted = 0 ORDER BY priority DESC")
    fun getActiveTasks(): Flow<List<Task>>

    @Query("SELECT * FROM Task WHERE priority = :priority")
    fun getByPriority(priority: Priority): Flow<List<Task>>

    @Query("UPDATE Task SET isCompleted = 1 WHERE id = :id")
    suspend fun complete(id: Long)
}
```

---

## まとめ

| 機能 | 用途 |
|------|------|
| `autoGenerate = true` | ID自動採番 |
| `defaultValue` | SQLデフォルト値 |
| `@TypeConverter` | 型変換定義 |
| `@TypeConverters` | DB全体に適用 |

- `autoGenerate`で`id = 0`にするとINSERT時に自動採番
- `defaultValue`はMigration時の既存行にも適用
- `TypeConverter`でDate/Enum/Listを永続化
- `@ColumnInfo`でカラム名やデフォルト値を指定

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Upsert](https://zenn.dev/myougatheaxo/articles/android-compose-room-upsert-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
