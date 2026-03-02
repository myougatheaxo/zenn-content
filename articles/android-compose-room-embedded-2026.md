---
title: "Room Embedded完全ガイド — @Embedded/複合主キー/インデックス/ビュー"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Embedded**（@Embedded、複合主キー、インデックス、DatabaseView）を解説します。

---

## @Embedded

```kotlin
data class Address(
    val street: String,
    val city: String,
    val postalCode: String,
    val country: String = "日本"
)

@Entity
data class Customer(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val email: String,
    @Embedded(prefix = "home_") val homeAddress: Address,
    @Embedded(prefix = "office_") val officeAddress: Address?
)

// SQLiteカラム: id, name, email, home_street, home_city, home_postalCode, home_country, office_street, ...

@Dao
interface CustomerDao {
    @Query("SELECT * FROM Customer WHERE home_city = :city")
    fun getByCity(city: String): Flow<List<Customer>>

    @Insert
    suspend fun insert(customer: Customer)
}
```

---

## 複合主キーとインデックス

```kotlin
@Entity(
    primaryKeys = ["date", "userId"],
    indices = [
        Index(value = ["userId"]),
        Index(value = ["date", "category"], unique = true)
    ]
)
data class DailyRecord(
    val date: String,
    val userId: Long,
    val category: String,
    val value: Double,
    @Embedded val metadata: RecordMetadata
)

data class RecordMetadata(
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis(),
    val source: String = "manual"
)
```

---

## DatabaseView

```kotlin
@DatabaseView("""
    SELECT u.userId, u.name,
           COUNT(p.postId) as postCount,
           MAX(p.createdAt) as lastPostDate
    FROM User u
    LEFT JOIN Post p ON u.userId = p.authorId
    GROUP BY u.userId
""")
data class UserSummary(
    val userId: Long,
    val name: String,
    val postCount: Int,
    val lastPostDate: Long?
)

@Database(
    entities = [User::class, Post::class],
    views = [UserSummary::class],
    version = 1
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userSummaryDao(): UserSummaryDao
}

@Dao
interface UserSummaryDao {
    @Query("SELECT * FROM UserSummary ORDER BY postCount DESC")
    fun getActiveUsers(): Flow<List<UserSummary>>
}
```

---

## まとめ

| 機能 | アノテーション |
|------|-------------|
| 埋め込み | `@Embedded` |
| プレフィクス | `prefix = "home_"` |
| 複合主キー | `primaryKeys = [...]` |
| ビュー | `@DatabaseView` |

- `@Embedded`でデータクラスをカラムに展開
- `prefix`で同型の複数Embedded区別
- `@Index`で検索パフォーマンス向上
- `@DatabaseView`でJOIN結果を仮想テーブル化

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room Relation](https://zenn.dev/myougatheaxo/articles/android-compose-room-relation-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
