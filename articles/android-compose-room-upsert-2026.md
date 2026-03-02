---
title: "Room Upsert完全ガイド — @Upsert/OnConflictStrategy/トランザクション"
emoji: "🔃"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Upsert**（@Upsert、OnConflictStrategy、@Transaction、バッチ操作）を解説します。

---

## @Upsert

```kotlin
@Dao
interface UserDao {
    // Room 2.5+: @Upsert = INSERT or UPDATE
    @Upsert
    suspend fun upsert(user: User)

    @Upsert
    suspend fun upsertAll(users: List<User>)

    // 従来のOnConflictStrategy
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertOrReplace(user: User)

    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertOrIgnore(user: User)
}

@Entity
data class User(
    @PrimaryKey val id: Long,
    val name: String,
    val email: String,
    val updatedAt: Long = System.currentTimeMillis()
)
```

---

## @Transaction

```kotlin
@Dao
abstract class OrderDao {
    @Upsert
    abstract suspend fun upsertOrder(order: Order)

    @Upsert
    abstract suspend fun upsertOrderItems(items: List<OrderItem>)

    @Query("DELETE FROM OrderItem WHERE orderId = :orderId")
    abstract suspend fun deleteOrderItems(orderId: Long)

    @Transaction
    open suspend fun upsertOrderWithItems(order: Order, items: List<OrderItem>) {
        upsertOrder(order)
        deleteOrderItems(order.id)
        upsertOrderItems(items)
    }
}
```

---

## API同期パターン

```kotlin
class UserRepository @Inject constructor(
    private val api: UserApi,
    private val dao: UserDao
) {
    suspend fun syncUsers() {
        val remoteUsers = api.getUsers()
        dao.upsertAll(remoteUsers.map { it.toEntity() })
    }

    fun getUsers(): Flow<List<User>> = dao.getAll()

    suspend fun updateUser(user: User) {
        dao.upsert(user.copy(updatedAt = System.currentTimeMillis()))
        // サーバーにも送信（オフラインキューで）
    }
}
```

---

## まとめ

| API | 動作 |
|-----|------|
| `@Upsert` | INSERT or UPDATE |
| `REPLACE` | DELETE+INSERT（IDリセット注意） |
| `IGNORE` | 重複時スキップ |
| `@Transaction` | アトミック操作 |

- `@Upsert`で既存レコードは更新、新規は挿入
- `REPLACE`はDELETE+INSERTなのでFK注意
- `@Transaction`で複数操作をアトミックに
- API同期パターンで`upsertAll`が活躍

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [オフラインファースト](https://zenn.dev/myougatheaxo/articles/android-compose-offline-first-2026)
