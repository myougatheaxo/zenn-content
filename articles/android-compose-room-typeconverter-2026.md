---
title: "Room TypeConverter完全ガイド — カスタム型/JSON/Enum/日付の永続化"
emoji: "🗄️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room TypeConverter**（カスタム型変換、JSON保存、Enum永続化、日付型、kotlinx-serialization連携）を解説します。

---

## 基本のTypeConverter

```kotlin
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? = date?.time
}

// Database に登録
@Database(entities = [Task::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao
}
```

---

## Enum変換

```kotlin
enum class Priority { LOW, MEDIUM, HIGH, URGENT }

class EnumConverters {
    @TypeConverter
    fun fromPriority(priority: Priority): String = priority.name

    @TypeConverter
    fun toPriority(value: String): Priority = Priority.valueOf(value)
}

@Entity
data class Task(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val title: String,
    val priority: Priority,
    val dueDate: Date?
)
```

---

## リスト/JSON変換（kotlinx-serialization）

```kotlin
class ListConverters {
    @TypeConverter
    fun fromStringList(value: List<String>): String {
        return Json.encodeToString(value)
    }

    @TypeConverter
    fun toStringList(value: String): List<String> {
        return Json.decodeFromString(value)
    }

    @TypeConverter
    fun fromTagList(value: List<Tag>): String {
        return Json.encodeToString(value)
    }

    @TypeConverter
    fun toTagList(value: String): List<Tag> {
        return Json.decodeFromString(value)
    }
}

@Serializable
data class Tag(val name: String, val color: String)

@Entity
data class Article(
    @PrimaryKey val id: String,
    val title: String,
    val tags: List<Tag>,       // JSONとして保存
    val keywords: List<String>  // JSONとして保存
)
```

---

## LocalDateTime変換

```kotlin
class DateTimeConverters {
    @TypeConverter
    fun fromLocalDateTime(value: LocalDateTime?): String? {
        return value?.toString()
    }

    @TypeConverter
    fun toLocalDateTime(value: String?): LocalDateTime? {
        return value?.let { LocalDateTime.parse(it) }
    }

    @TypeConverter
    fun fromLocalDate(value: LocalDate?): String? = value?.toString()

    @TypeConverter
    fun toLocalDate(value: String?): LocalDate? = value?.let { LocalDate.parse(it) }
}
```

---

## 埋め込みオブジェクト（Embedded）

```kotlin
// TypeConverterの代わりに@Embeddedを使う方法
data class Address(
    val street: String,
    val city: String,
    val zipCode: String
)

@Entity
data class User(
    @PrimaryKey val id: String,
    val name: String,
    @Embedded(prefix = "addr_") val address: Address
)
// → addr_street, addr_city, addr_zipCode カラムが生成される
```

---

## まとめ

| 型 | 変換方法 |
|----|---------|
| Date | Long (timestamp) |
| Enum | String (name) |
| List | JSON文字列 |
| LocalDateTime | ISO文字列 |
| 複合型 | `@Embedded` |

- `@TypeConverter`でカスタム型をRoom対応
- `@TypeConverters`をDatabaseクラスに登録
- JSON変換はkotlinx-serializationが便利
- `@Embedded`は別テーブル不要の埋め込み

---

8種類のAndroidアプリテンプレート（Room設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room CRUD](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-crud-2026)
- [Roomマイグレーション](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-2026)
- [Room FTS](https://zenn.dev/myougatheaxo/articles/android-compose-room-fts-2026)
