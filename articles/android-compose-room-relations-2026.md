---
title: "Room リレーション完全ガイド — 1対多/多対多/Embedded"
emoji: "🔗"
type: "tech"
topics: ["android", "kotlin", "room", "database"]
published: true
---

## この記事で学べること

**Room**のリレーション（1対多、多対多、Embedded、Transaction）を解説します。

---

## 1対1 (@Embedded)

```kotlin
@Entity
data class User(
    @PrimaryKey val userId: String,
    val name: String
)

data class Address(
    val street: String,
    val city: String,
    val postalCode: String
)

// Embeddedでフラットに格納
@Entity
data class UserWithAddress(
    @PrimaryKey val userId: String,
    val name: String,
    @Embedded val address: Address
)
// テーブル: userId, name, street, city, postalCode
```

---

## 1対多 (@Relation)

```kotlin
@Entity
data class Author(
    @PrimaryKey val authorId: String,
    val name: String
)

@Entity
data class Book(
    @PrimaryKey val bookId: String,
    val title: String,
    val authorId: String // FK
)

// 結果クラス（Entityではない）
data class AuthorWithBooks(
    @Embedded val author: Author,
    @Relation(
        parentColumn = "authorId",
        entityColumn = "authorId"
    )
    val books: List<Book>
)

@Dao
interface AuthorDao {
    @Transaction
    @Query("SELECT * FROM Author")
    fun getAuthorsWithBooks(): Flow<List<AuthorWithBooks>>

    @Transaction
    @Query("SELECT * FROM Author WHERE authorId = :id")
    suspend fun getAuthorWithBooks(id: String): AuthorWithBooks?
}
```

---

## 多対多

```kotlin
@Entity
data class Student(
    @PrimaryKey val studentId: String,
    val name: String
)

@Entity
data class Course(
    @PrimaryKey val courseId: String,
    val title: String
)

// 中間テーブル
@Entity(primaryKeys = ["studentId", "courseId"])
data class StudentCourseCrossRef(
    val studentId: String,
    val courseId: String
)

// 結果クラス
data class StudentWithCourses(
    @Embedded val student: Student,
    @Relation(
        parentColumn = "studentId",
        entityColumn = "courseId",
        associateBy = Junction(StudentCourseCrossRef::class)
    )
    val courses: List<Course>
)

@Dao
interface StudentDao {
    @Transaction
    @Query("SELECT * FROM Student")
    fun getStudentsWithCourses(): Flow<List<StudentWithCourses>>

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun enrollCourse(crossRef: StudentCourseCrossRef)
}
```

---

## TypeConverter

```kotlin
class Converters {
    @TypeConverter
    fun fromStringList(value: List<String>): String = value.joinToString(",")

    @TypeConverter
    fun toStringList(value: String): List<String> =
        if (value.isEmpty()) emptyList() else value.split(",")

    @TypeConverter
    fun fromDate(date: Date?): Long? = date?.time

    @TypeConverter
    fun toDate(timestamp: Long?): Date? = timestamp?.let { Date(it) }
}

@Database(
    entities = [User::class, Book::class],
    version = 1
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase()
```

---

## Transaction

```kotlin
@Dao
abstract class OrderDao {
    @Insert
    abstract suspend fun insertOrder(order: Order): Long

    @Insert
    abstract suspend fun insertOrderItems(items: List<OrderItem>)

    @Transaction
    open suspend fun createOrderWithItems(order: Order, items: List<OrderItem>) {
        val orderId = insertOrder(order)
        val itemsWithOrderId = items.map { it.copy(orderId = orderId) }
        insertOrderItems(itemsWithOrderId)
    }
}
```

---

## まとめ

- `@Embedded`で1対1のフラット格納
- `@Relation`で1対多のリレーション
- `Junction`で多対多の中間テーブル
- `@Transaction`でリレーション取得時の一貫性保証
- `@TypeConverters`でカスタム型の変換
- リレーション結果クラスはEntityではなくdata class

---

8種類のAndroidアプリテンプレート（Room DB設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room マイグレーション](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-2026)
- [Paging3 + Room](https://zenn.dev/myougatheaxo/articles/android-compose-paging-room-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
