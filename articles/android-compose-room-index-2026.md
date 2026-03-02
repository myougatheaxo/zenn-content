---
title: "Room Index完全ガイド — インデックス/複合インデックス/ユニーク制約"
emoji: "📇"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Index**（@Index、複合インデックス、ユニーク制約、クエリ最適化）を解説します。

---

## 基本Index

```kotlin
@Entity(
    indices = [
        Index("email", unique = true),
        Index("name"),
        Index("createdAt")
    ]
)
data class User(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val email: String,
    val createdAt: Long = System.currentTimeMillis()
)

// emailでの検索が高速化
@Dao
interface UserDao {
    @Query("SELECT * FROM User WHERE email = :email")
    suspend fun getByEmail(email: String): User?

    @Query("SELECT * FROM User ORDER BY createdAt DESC")
    fun getAll(): Flow<List<User>>
}
```

---

## 複合Index

```kotlin
@Entity(
    indices = [
        Index("authorId", "createdAt"),  // 著者の記事を日付順に取得
        Index("category", "publishedAt", unique = false)
    ]
)
data class Article(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val authorId: Long,
    val category: String,
    val createdAt: Long = System.currentTimeMillis(),
    val publishedAt: Long? = null
)

@Dao
interface ArticleDao {
    // 複合インデックスが効く
    @Query("SELECT * FROM Article WHERE authorId = :authorId ORDER BY createdAt DESC")
    fun getByAuthor(authorId: Long): Flow<List<Article>>

    @Query("SELECT * FROM Article WHERE category = :category ORDER BY publishedAt DESC")
    fun getByCategory(category: String): Flow<List<Article>>
}
```

---

## ForeignKey + Index

```kotlin
@Entity(
    foreignKeys = [
        ForeignKey(
            entity = User::class,
            parentColumns = ["id"],
            childColumns = ["userId"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index("userId")]  // ForeignKeyにはindexが推奨
)
data class Order(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val userId: Long,
    val total: Double,
    val createdAt: Long = System.currentTimeMillis()
)
```

---

## まとめ

| 設定 | 用途 |
|------|------|
| `@Index` | 単一カラムインデックス |
| 複合Index | 複数カラムインデックス |
| `unique = true` | ユニーク制約 |
| ForeignKey Index | 外部キー最適化 |

- `@Index`でクエリのWHERE/ORDER BYを高速化
- 複合インデックスは左端のカラムから有効
- `unique = true`でデータ一意性を保証
- ForeignKeyカラムにはインデックスが推奨

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room FTS4](https://zenn.dev/myougatheaxo/articles/android-compose-room-fts4-2026)
- [Room Relation](https://zenn.dev/myougatheaxo/articles/android-compose-room-relation-2026)
- [Room TypeConverter](https://zenn.dev/myougatheaxo/articles/android-compose-room-type-converter-2026)
