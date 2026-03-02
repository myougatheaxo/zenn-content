---
title: "Kotlin Value Class完全ガイド — @JvmInline/型安全ID/ゼロコスト抽象化"
emoji: "💎"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "valueclass"]
published: true
---

## この記事で学べること

**Kotlin Value Class**（@JvmInline value class、型安全ラッパー、ゼロコスト抽象化、Compose連携）を解説します。

---

## 基本

```kotlin
// ❌ String型のIDは混同しやすい
fun getUser(userId: String, teamId: String): User // 引数を逆にしてもコンパイルエラーにならない

// ✅ Value Classで型安全に
@JvmInline value class UserId(val value: String)
@JvmInline value class TeamId(val value: String)

fun getUser(userId: UserId, teamId: TeamId): User
// getUser(teamId, userId) → コンパイルエラー！
```

---

## 実践パターン

```kotlin
// ID型
@JvmInline value class ArticleId(val value: String)
@JvmInline value class CategoryId(val value: Int)

// 金額型（単位の混同防止）
@JvmInline value class Yen(val amount: Int) {
    operator fun plus(other: Yen) = Yen(amount + other.amount)
    operator fun times(quantity: Int) = Yen(amount * quantity)
    fun formatted(): String = "¥${"%,d".format(amount)}"
}

// メールアドレス（バリデーション付き）
@JvmInline value class Email(val value: String) {
    init {
        require(value.contains("@")) { "無効なメールアドレス: $value" }
    }
}

// 使用
val price = Yen(1000)
val total = price * 3  // Yen(3000)
println(total.formatted()) // ¥3,000
```

---

## Compose連携

```kotlin
@JvmInline value class ItemId(val value: String)

@Composable
fun ItemDetailScreen(itemId: ItemId) {
    // 型安全なIDで誤った引数を防止
    val viewModel: ItemDetailViewModel = hiltViewModel()

    LaunchedEffect(itemId) {
        viewModel.loadItem(itemId)
    }
}

// Navigation
composable("item/{id}",
    arguments = listOf(navArgument("id") { type = NavType.StringType })
) { backStackEntry ->
    val id = ItemId(backStackEntry.arguments?.getString("id") ?: "")
    ItemDetailScreen(itemId = id)
}

// Repository
class ItemRepository @Inject constructor(private val dao: ItemDao) {
    suspend fun getItem(id: ItemId): Item = dao.getById(id.value)
    suspend fun deleteItem(id: ItemId) = dao.deleteById(id.value)
}
```

---

## まとめ

| 特徴 | 説明 |
|------|------|
| ゼロコスト | ランタイムではプリミティブに展開 |
| 型安全 | コンパイル時に型チェック |
| `init` | バリデーション追加可能 |
| `operator` | 演算子オーバーロード |

- Value Classはコンパイル後にアンラップされる（ゼロコスト）
- ID/金額/メールなどのドメイン型に最適
- `init`ブロックでバリデーション追加
- `@JvmInline`アノテーションが必須

---

8種類のAndroidアプリテンプレート（Kotlin対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin SealedClass](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-sealed-class-2026)
- [Kotlin Delegation](https://zenn.dev/myougatheaxo/articles/android-compose-kotlin-delegation-2026)
- [Compose CleanArchitecture](https://zenn.dev/myougatheaxo/articles/android-compose-compose-clean-architecture-2026)
