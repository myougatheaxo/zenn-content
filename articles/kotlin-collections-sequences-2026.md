---
title: "コレクション操作ガイド — Kotlinのmap/filter/groupByを使いこなす"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "collections", "functional"]
published: true
---

## この記事で学べること

Kotlinの**コレクション操作**（map, filter, groupBy等）と**Sequence**の使い分けを解説します。

---

## 変換: map / flatMap

```kotlin
val names = listOf("Alice", "Bob", "Charlie")

// map: 1対1変換
val lengths = names.map { it.length } // [5, 3, 7]

// mapIndexed: インデックス付き
val indexed = names.mapIndexed { i, name -> "${i + 1}. $name" }

// mapNotNull: null除外
val numbers = listOf("1", "two", "3")
val parsed = numbers.mapNotNull { it.toIntOrNull() } // [1, 3]

// flatMap: リストのリストを平坦化
data class Order(val items: List<String>)
val orders = listOf(
    Order(listOf("A", "B")),
    Order(listOf("C"))
)
val allItems = orders.flatMap { it.items } // [A, B, C]
```

---

## フィルタリング

```kotlin
data class User(val name: String, val age: Int, val active: Boolean)

val users = listOf(
    User("Alice", 28, true),
    User("Bob", 35, false),
    User("Charlie", 22, true)
)

// filter
val activeUsers = users.filter { it.active }

// filterNot
val inactiveUsers = users.filterNot { it.active }

// partition: 2グループに分割
val (active, inactive) = users.partition { it.active }

// first / firstOrNull
val adult = users.first { it.age >= 30 }
val teen = users.firstOrNull { it.age < 20 } // null
```

---

## グルーピング・集計

```kotlin
data class Product(val category: String, val name: String, val price: Int)

val products = listOf(
    Product("食品", "りんご", 200),
    Product("食品", "バナナ", 150),
    Product("家電", "充電器", 1500),
    Product("家電", "ケーブル", 800)
)

// groupBy
val byCategory = products.groupBy { it.category }
// {食品=[りんご, バナナ], 家電=[充電器, ケーブル]}

// associateBy: キーでマップ化（重複時は上書き）
val byName = products.associateBy { it.name }

// sumOf
val total = products.sumOf { it.price } // 2650

// maxByOrNull / minByOrNull
val expensive = products.maxByOrNull { it.price }

// count
val foodCount = products.count { it.category == "食品" } // 2
```

---

## ソート

```kotlin
val items = listOf(3, 1, 4, 1, 5, 9)

// sorted（自然順）
val asc = items.sorted() // [1, 1, 3, 4, 5, 9]
val desc = items.sortedDescending()

// sortedBy（プロパティで）
val byAge = users.sortedBy { it.age }
val byAgeDesc = users.sortedByDescending { it.age }

// sortedWith（複合ソート）
val sorted = users.sortedWith(
    compareBy<User> { it.active }.thenByDescending { it.age }
)
```

---

## チェーン操作

```kotlin
data class Task(
    val title: String,
    val priority: Int,
    val completed: Boolean
)

fun getTopTasks(tasks: List<Task>): List<String> {
    return tasks
        .filter { !it.completed }
        .sortedByDescending { it.priority }
        .take(5)
        .map { it.title }
}
```

---

## Sequence（遅延評価）

```kotlin
// 大きなリストではSequenceが効率的
val result = (1..1_000_000)
    .asSequence()          // Sequence変換
    .filter { it % 2 == 0 }
    .map { it * it }
    .take(10)
    .toList()              // ここで初めて実行

// generateSequence
val powers = generateSequence(1) { it * 2 }
    .take(10)
    .toList() // [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]
```

### List vs Sequence の使い分け

```kotlin
// List: 小さいコレクション（数百件以下）
// → 各操作で中間リストが作られるが、小さいので問題なし
val small = listOf(1, 2, 3).filter { it > 1 }.map { it * 2 }

// Sequence: 大きいコレクション or チェーンが長い
// → 1要素ずつ全操作を通すので中間リスト不要
val big = (1..100000).asSequence()
    .filter { it % 3 == 0 }
    .map { it.toString() }
    .take(100)
    .toList()
```

---

## 実践: Composeでの活用

```kotlin
@Composable
fun CategoryList(products: List<Product>) {
    val grouped = remember(products) {
        products
            .filter { it.price > 0 }
            .groupBy { it.category }
            .toSortedMap()
    }

    LazyColumn {
        grouped.forEach { (category, items) ->
            stickyHeader {
                Text(
                    "$category (${items.size}件)",
                    Modifier.fillMaxWidth().background(Color.LightGray).padding(8.dp),
                    fontWeight = FontWeight.Bold
                )
            }
            items(items, key = { it.name }) { product ->
                ListItem(
                    headlineContent = { Text(product.name) },
                    trailingContent = { Text("¥${product.price}") }
                )
            }
        }
    }
}
```

---

## まとめ

- `map`/`flatMap`で変換、`mapNotNull`でnull除外
- `filter`/`partition`でフィルタリング
- `groupBy`でグルーピング、`associateBy`でMap化
- `sortedBy`/`sortedWith`でソート
- 大量データは`asSequence()`で遅延評価
- Composeでは`remember`内で計算しリコンポジション最適化

---

8種類のAndroidアプリテンプレート（効率的なデータ処理設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin data classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-2026)
- [Kotlin null安全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-null-safety-2026)
- [スコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
