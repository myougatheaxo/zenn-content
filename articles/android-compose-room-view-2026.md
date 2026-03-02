---
title: "Room DatabaseView完全ガイド — @DatabaseView/仮想テーブル/集約ビュー"
emoji: "👁️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room DatabaseView**（@DatabaseView、仮想テーブル、集約ビュー、JOIN結果のビュー化）を解説します。

---

## @DatabaseView基本

```kotlin
@DatabaseView("""
    SELECT u.id, u.name, u.email,
           COUNT(o.id) as orderCount,
           COALESCE(SUM(o.total), 0) as totalSpent
    FROM User u
    LEFT JOIN `Order` o ON u.id = o.userId
    GROUP BY u.id
""")
data class UserSummary(
    val id: Long,
    val name: String,
    val email: String,
    val orderCount: Int,
    val totalSpent: Double
)

@Database(
    entities = [User::class, Order::class],
    views = [UserSummary::class],
    version = 1
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

@Dao
interface UserDao {
    @Query("SELECT * FROM UserSummary ORDER BY totalSpent DESC")
    fun getUserSummaries(): Flow<List<UserSummary>>

    @Query("SELECT * FROM UserSummary WHERE id = :userId")
    suspend fun getUserSummary(userId: Long): UserSummary?
}
```

---

## 集約ビュー

```kotlin
@DatabaseView("""
    SELECT category,
           COUNT(*) as articleCount,
           MAX(publishedAt) as latestPublishedAt
    FROM Article
    WHERE publishedAt IS NOT NULL
    GROUP BY category
""")
data class CategoryStats(
    val category: String,
    val articleCount: Int,
    val latestPublishedAt: Long
)

@Dao
interface StatsDao {
    @Query("SELECT * FROM CategoryStats ORDER BY articleCount DESC")
    fun getCategoryStats(): Flow<List<CategoryStats>>
}
```

---

## Compose表示

```kotlin
@Composable
fun UserSummaryScreen(viewModel: UserViewModel = hiltViewModel()) {
    val summaries by viewModel.userSummaries.collectAsStateWithLifecycle()

    LazyColumn(Modifier.padding(16.dp)) {
        items(summaries) { summary ->
            Card(Modifier.fillMaxWidth().padding(vertical = 4.dp)) {
                ListItem(
                    headlineContent = { Text(summary.name) },
                    supportingContent = { Text("注文: ${summary.orderCount}件") },
                    trailingContent = { Text("¥${summary.totalSpent.toInt()}") }
                )
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@DatabaseView` | 仮想テーブル定義 |
| `views = [...]` | DB登録 |
| JOIN/GROUP BY | 複雑なクエリ |
| Flow | リアクティブ監視 |

- `@DatabaseView`で複雑なクエリをビュー化
- JOINや集約結果をシンプルなDAOで取得
- DBに登録が必要（`views`パラメータ）
- パフォーマンス注意：ビューは毎回クエリ実行

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Multimap](https://zenn.dev/myougatheaxo/articles/android-compose-room-multimap-2026)
- [Room Relation](https://zenn.dev/myougatheaxo/articles/android-compose-room-relation-2026)
- [Room Index](https://zenn.dev/myougatheaxo/articles/android-compose-room-index-2026)
