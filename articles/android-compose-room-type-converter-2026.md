---
title: "Room TypeConverter完全ガイド — カスタム型変換/JSON/Date/Enum"
emoji: "🔧"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room TypeConverter**（Date変換、JSON変換、Enum変換、List/Map変換）を解説します。

---

## 基本TypeConverter

```kotlin
class DateConverters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? = value?.let { Date(it) }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? = date?.time

    @TypeConverter
    fun fromLocalDate(value: String?): LocalDate? = value?.let { LocalDate.parse(it) }

    @TypeConverter
    fun localDateToString(date: LocalDate?): String? = date?.toString()

    @TypeConverter
    fun fromInstant(value: Long?): Instant? = value?.let { Instant.ofEpochMilli(it) }

    @TypeConverter
    fun instantToLong(instant: Instant?): Long? = instant?.toEpochMilli()
}
```

---

## JSON変換（kotlinx.serialization）

```kotlin
class JsonConverters {
    private val json = Json { ignoreUnknownKeys = true }

    @TypeConverter
    fun fromStringList(value: String): List<String> =
        json.decodeFromString(value)

    @TypeConverter
    fun stringListToJson(list: List<String>): String =
        json.encodeToString(list)

    @TypeConverter
    fun fromIntMap(value: String): Map<String, Int> =
        json.decodeFromString(value)

    @TypeConverter
    fun intMapToJson(map: Map<String, Int>): String =
        json.encodeToString(map)
}

@Entity
data class Recipe(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val ingredients: List<String>,  // JSON文字列に変換
    val nutritionFacts: Map<String, Int>  // JSON文字列に変換
)
```

---

## DB全体に適用

```kotlin
@Database(
    entities = [User::class, Recipe::class, Task::class],
    version = 1
)
@TypeConverters(DateConverters::class, JsonConverters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
    abstract fun recipeDao(): RecipeDao
    abstract fun taskDao(): TaskDao
}
```

---

## まとめ

| 変換元 | 変換先 | 用途 |
|--------|--------|------|
| `Date` | `Long` | タイムスタンプ |
| `LocalDate` | `String` | ISO日付文字列 |
| `List<T>` | `String(JSON)` | リスト保存 |
| `Enum` | `String` | 列挙値保存 |

- `@TypeConverter`でRoom非対応型をサポート型に変換
- Date系はLongまたはString保存
- List/MapはJSON文字列に変換
- `@TypeConverters`でDB全体に適用

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room AutoGenerate](https://zenn.dev/myougatheaxo/articles/android-compose-room-auto-value-2026)
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
