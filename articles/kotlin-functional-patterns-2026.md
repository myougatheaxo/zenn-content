---
title: "Kotlin関数型パターンガイド — 高階関数/ラムダ/インライン"
emoji: "λ"
type: "tech"
topics: ["android", "kotlin", "functional", "patterns"]
published: true
---

## この記事で学べること

Kotlinの**関数型プログラミングパターン**（高階関数、ラムダ、インライン関数）を解説します。

---

## 高階関数

```kotlin
// 関数を引数に受け取る
fun <T> List<T>.customFilter(predicate: (T) -> Boolean): List<T> {
    val result = mutableListOf<T>()
    for (item in this) {
        if (predicate(item)) result.add(item)
    }
    return result
}

val numbers = listOf(1, 2, 3, 4, 5)
val evens = numbers.customFilter { it % 2 == 0 } // [2, 4]

// 関数を返す
fun multiplier(factor: Int): (Int) -> Int = { it * factor }
val double = multiplier(2)
val triple = multiplier(3)
println(double(5))  // 10
println(triple(5))  // 15
```

---

## ラムダと関数参照

```kotlin
// ラムダ式
val sum: (Int, Int) -> Int = { a, b -> a + b }

// 関数参照
fun isEven(n: Int) = n % 2 == 0
val numbers = listOf(1, 2, 3, 4, 5)
numbers.filter(::isEven) // [2, 4]

// メソッド参照
val strings = listOf("hello", "world")
strings.map(String::uppercase) // [HELLO, WORLD]

// コンストラクタ参照
data class User(val name: String)
val names = listOf("Alice", "Bob")
val users = names.map(::User) // [User(Alice), User(Bob)]
```

---

## inline関数

```kotlin
// inline: ラムダのオーバーヘッドを除去
inline fun <T> measureTime(block: () -> T): Pair<T, Long> {
    val start = System.currentTimeMillis()
    val result = block()
    val elapsed = System.currentTimeMillis() - start
    return result to elapsed
}

val (data, time) = measureTime {
    repository.fetchData()
}

// crossinline: ラムダ内でreturnを禁止
inline fun runSafely(crossinline block: () -> Unit) {
    try {
        block()
    } catch (e: Exception) {
        // エラーログ
    }
}

// noinline: 特定のラムダをインライン化しない
inline fun execute(
    noinline onComplete: () -> Unit, // 変数として保存する必要があるため
    block: () -> Unit
) {
    block()
    callbacks.add(onComplete)
}
```

---

## 関数の合成

```kotlin
// 関数合成
infix fun <A, B, C> ((B) -> C).compose(f: (A) -> B): (A) -> C = { a -> this(f(a)) }

val addOne: (Int) -> Int = { it + 1 }
val double: (Int) -> Int = { it * 2 }
val doubleThenAddOne = addOne compose double
println(doubleThenAddOne(3)) // 7 (3*2 + 1)

// パイプライン
fun String.pipeline(vararg transforms: (String) -> String): String {
    return transforms.fold(this) { acc, transform -> transform(acc) }
}

val result = "  Hello World  ".pipeline(
    String::trim,
    String::lowercase,
    { it.replace(" ", "-") }
)
// "hello-world"
```

---

## Android実践

```kotlin
// ViewModelでの高階関数活用
class ItemViewModel : ViewModel() {
    fun loadItems(
        filter: (Item) -> Boolean = { true },
        sort: Comparator<Item> = compareBy { it.createdAt },
        transform: (Item) -> Item = { it }
    ) {
        viewModelScope.launch {
            val items = repository.getAll()
                .filter(filter)
                .sortedWith(sort)
                .map(transform)
            _items.value = items
        }
    }
}

// Compose: 再利用可能なコンポーネント
@Composable
fun <T> GenericList(
    items: List<T>,
    key: (T) -> Any,
    itemContent: @Composable (T) -> Unit,
    emptyContent: @Composable () -> Unit = { Text("データなし") }
) {
    if (items.isEmpty()) {
        emptyContent()
    } else {
        LazyColumn {
            items(items, key = key) { item ->
                itemContent(item)
            }
        }
    }
}
```

---

## まとめ

- 高階関数: 関数を引数/戻り値として扱う
- ラムダ式と関数参照(`::`)で簡潔に記述
- `inline`でラムダのオーバーヘッド除去
- `crossinline`/`noinline`で制御
- 関数合成でパイプライン処理
- Compose: `@Composable`ラムダで柔軟なUI設計

---

8種類のAndroidアプリテンプレート（関数型パターン活用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [スコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [拡張関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-extension-functions-2026)
- [Collections/Sequencesガイド](https://zenn.dev/myougatheaxo/articles/kotlin-collections-sequences-2026)
