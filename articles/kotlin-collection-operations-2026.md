---
title: "Kotlinコレクション操作ガイド — map/filter/groupBy/zip"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "collection", "tips"]
published: true
---

## この記事で学べること

Kotlinの**コレクション操作**（変換、フィルタ、集約、グループ化）を解説します。

---

## 変換: map / flatMap

```kotlin
data class User(val name: String, val tags: List<String>)

val users = listOf(
    User("Alice", listOf("kotlin", "android")),
    User("Bob", listOf("swift", "ios"))
)

// map: 要素を変換
val names = users.map { it.name } // ["Alice", "Bob"]

// mapNotNull: nullを除外
val emails = users.mapNotNull { it.email } // nullはスキップ

// flatMap: ネストを展開
val allTags = users.flatMap { it.tags } // ["kotlin", "android", "swift", "ios"]

// mapIndexed: インデックス付き
val numbered = names.mapIndexed { i, name -> "${i + 1}. $name" }
```

---

## フィルタ: filter / partition

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// filter
val evens = numbers.filter { it % 2 == 0 } // [2, 4, 6, 8, 10]

// filterNot
val odds = numbers.filterNot { it % 2 == 0 }

// partition: 条件でtrue/falseに分割
val (big, small) = numbers.partition { it > 5 }
// big = [6, 7, 8, 9, 10], small = [1, 2, 3, 4, 5]

// filterIsInstance: 型でフィルタ
val mixed: List<Any> = listOf(1, "two", 3, "four")
val strings = mixed.filterIsInstance<String>() // ["two", "four"]
```

---

## 集約: reduce / fold

```kotlin
val prices = listOf(100, 200, 150, 300)

// sum
val total = prices.sum() // 750

// sumOf
val totalPrice = items.sumOf { it.price }

// reduce: 初期値なし
val product = listOf(1, 2, 3, 4).reduce { acc, i -> acc * i } // 24

// fold: 初期値あり
val csv = names.fold("") { acc, name ->
    if (acc.isEmpty()) name else "$acc, $name"
}

// maxBy / minBy
val mostExpensive = items.maxBy { it.price }
```

---

## グループ化: groupBy / associateBy

```kotlin
data class Task(val category: String, val title: String, val done: Boolean)

val tasks = listOf(
    Task("Work", "レビュー", false),
    Task("Work", "実装", true),
    Task("Personal", "買い物", false)
)

// groupBy: キーでグループ化
val byCategory = tasks.groupBy { it.category }
// {"Work": [Task, Task], "Personal": [Task]}

// associateBy: キーで1対1マッピング
val byTitle = tasks.associateBy { it.title }
// {"レビュー": Task, "実装": Task, "買い物": Task}

// associate: 任意のキー値ペア
val titleToStatus = tasks.associate { it.title to it.done }
// {"レビュー": false, "実装": true, "買い物": false}

// chunked: 固定サイズに分割
val batches = (1..10).chunked(3) // [[1,2,3], [4,5,6], [7,8,9], [10]]

// windowed: スライディングウィンドウ
val windows = (1..5).windowed(3) // [[1,2,3], [2,3,4], [3,4,5]]
```

---

## zip / unzip

```kotlin
val names = listOf("Alice", "Bob", "Charlie")
val ages = listOf(25, 30, 35)

// zip: 2つのリストを結合
val pairs = names.zip(ages) // [(Alice, 25), (Bob, 30), (Charlie, 35)]

// zip + transform
val descriptions = names.zip(ages) { name, age -> "$name ($age歳)" }

// unzip: ペアリストを2つに分割
val (nameList, ageList) = pairs.unzip()
```

---

## まとめ

- `map`/`flatMap`で変換、`mapNotNull`でnull除外
- `filter`/`partition`で条件分割
- `fold`/`reduce`で集約、`sumOf`で合計
- `groupBy`でグループ化、`associateBy`でマップ化
- `chunked`でバッチ分割、`windowed`でスライディング
- `zip`/`unzip`でリスト結合/分割

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin関数型パターン](https://zenn.dev/myougatheaxo/articles/kotlin-functional-patterns-2026)
- [スコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [Flow+Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-flow-lifecycle-2026)
