---
title: "Room RawQuery完全ガイド — @RawQuery/動的クエリ/SupportSQLiteQuery"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "room"]
published: true
---

## この記事で学べること

**Room RawQuery**（@RawQuery、動的クエリ生成、SupportSQLiteQuery、フィルター検索）を解説します。

---

## @RawQuery

```kotlin
@Dao
interface ProductDao {
    @RawQuery(observedEntities = [Product::class])
    fun getProducts(query: SupportSQLiteQuery): Flow<List<Product>>

    @RawQuery
    suspend fun getProductCount(query: SupportSQLiteQuery): Int
}
```

---

## 動的フィルタークエリ

```kotlin
class ProductRepository @Inject constructor(private val dao: ProductDao) {

    fun searchProducts(
        keyword: String?,
        category: String?,
        minPrice: Double?,
        maxPrice: Double?,
        sortBy: String = "name",
        sortOrder: String = "ASC"
    ): Flow<List<Product>> {
        val conditions = mutableListOf<String>()
        val args = mutableListOf<Any>()

        keyword?.let {
            conditions.add("(name LIKE ? OR description LIKE ?)")
            args.add("%$it%"); args.add("%$it%")
        }
        category?.let {
            conditions.add("category = ?")
            args.add(it)
        }
        minPrice?.let {
            conditions.add("price >= ?")
            args.add(it)
        }
        maxPrice?.let {
            conditions.add("price <= ?")
            args.add(it)
        }

        val where = if (conditions.isNotEmpty())
            "WHERE ${conditions.joinToString(" AND ")}" else ""

        val sql = "SELECT * FROM Product $where ORDER BY $sortBy $sortOrder"
        val query = SimpleSQLiteQuery(sql, args.toTypedArray())
        return dao.getProducts(query)
    }
}
```

---

## ViewModel + Compose

```kotlin
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val repository: ProductRepository
) : ViewModel() {
    private val _filters = MutableStateFlow(ProductFilter())
    val filters = _filters.asStateFlow()

    val products = _filters
        .flatMapLatest { f ->
            repository.searchProducts(
                keyword = f.keyword,
                category = f.category,
                minPrice = f.minPrice,
                maxPrice = f.maxPrice,
                sortBy = f.sortBy
            )
        }
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())

    fun updateFilter(filter: ProductFilter) { _filters.value = filter }
}

data class ProductFilter(
    val keyword: String? = null,
    val category: String? = null,
    val minPrice: Double? = null,
    val maxPrice: Double? = null,
    val sortBy: String = "name"
)
```

---

## まとめ

| API | 用途 |
|-----|------|
| `@RawQuery` | 動的SQLクエリ |
| `SimpleSQLiteQuery` | クエリ生成 |
| `observedEntities` | Flowの監視対象 |
| パラメータバインド | SQLインジェクション防止 |

- `@RawQuery`で実行時にSQLを動的生成
- `SimpleSQLiteQuery`でパラメータバインド（安全）
- `observedEntities`でFlowの監視テーブル指定
- 複雑なフィルター検索に最適

---

8種類のAndroidアプリテンプレート（Room DB対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room/Flow](https://zenn.dev/myougatheaxo/articles/android-compose-room-flow-2026)
- [Room FTS4](https://zenn.dev/myougatheaxo/articles/android-compose-room-fts4-2026)
- [Room Paging](https://zenn.dev/myougatheaxo/articles/android-compose-room-paging-2026)
