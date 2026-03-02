---
title: "Room Transaction完全ガイド — @Transaction/一括操作/整合性保証"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room Transaction**（@Transaction、一括操作、データ整合性、ネストトランザクション）を解説します。

---

## 基本@Transaction

```kotlin
@Dao
interface OrderDao {
    @Insert
    suspend fun insertOrder(order: Order): Long

    @Insert
    suspend fun insertOrderItems(items: List<OrderItem>)

    @Query("UPDATE Product SET stock = stock - :quantity WHERE id = :productId")
    suspend fun reduceStock(productId: Long, quantity: Int)

    @Transaction
    suspend fun placeOrder(order: Order, items: List<OrderItem>) {
        val orderId = insertOrder(order)
        val itemsWithOrderId = items.map { it.copy(orderId = orderId) }
        insertOrderItems(itemsWithOrderId)
        items.forEach { reduceStock(it.productId, it.quantity) }
    }
}
```

---

## Relation取得

```kotlin
data class UserWithOrders(
    @Embedded val user: User,
    @Relation(parentColumn = "id", entityColumn = "userId")
    val orders: List<Order>
)

@Dao
interface UserDao {
    // @Relationクエリには@Transaction必須
    @Transaction
    @Query("SELECT * FROM User WHERE id = :userId")
    suspend fun getUserWithOrders(userId: Long): UserWithOrders

    @Transaction
    @Query("SELECT * FROM User")
    fun getAllUsersWithOrders(): Flow<List<UserWithOrders>>
}
```

---

## 一括更新/削除

```kotlin
@Dao
interface TaskDao {
    @Query("DELETE FROM Task WHERE id IN (:ids)")
    suspend fun deleteByIds(ids: List<Long>)

    @Query("UPDATE Task SET completed = 1 WHERE id IN (:ids)")
    suspend fun completeByIds(ids: List<Long>)

    @Transaction
    suspend fun archiveCompleted() {
        val completed = getCompletedTasks()
        insertArchive(completed.map { Archive(taskId = it.id, title = it.title) })
        deleteByIds(completed.map { it.id })
    }

    @Query("SELECT * FROM Task WHERE completed = 1")
    suspend fun getCompletedTasks(): List<Task>

    @Insert
    suspend fun insertArchive(archives: List<Archive>)
}
```

---

## まとめ

| 用途 | 方法 |
|------|------|
| 複数テーブル操作 | `@Transaction`メソッド |
| Relation取得 | `@Transaction` + `@Query` |
| 一括更新/削除 | `@Transaction`で整合性保証 |
| ロールバック | 例外発生で自動ロールバック |

- `@Transaction`で複数操作をアトミックに実行
- `@Relation`を含むクエリには`@Transaction`必須
- 例外発生時は全操作が自動ロールバック
- suspend関数内で自然に使用可能

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Relation](https://zenn.dev/myougatheaxo/articles/android-compose-room-relation-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
- [Room Paging](https://zenn.dev/myougatheaxo/articles/android-compose-room-paging-2026)
