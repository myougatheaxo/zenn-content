---
title: "Kotlinジェネリクス入門 — 型パラメータ・共変・反変を理解する"
emoji: "🔷"
type: "tech"
topics: ["android", "kotlin", "generics", "type"]
published: true
---

## この記事で学べること

Kotlinの**ジェネリクス**（型パラメータ、共変・反変、reified）を解説します。

---

## ジェネリクスの基本

```kotlin
// ジェネリッククラス
class Box<T>(val value: T)

val intBox = Box(42)        // Box<Int>
val strBox = Box("hello")   // Box<String>

// ジェネリック関数
fun <T> singletonList(item: T): List<T> = listOf(item)

// 型制約
fun <T : Comparable<T>> sort(list: List<T>): List<T> {
    return list.sorted()
}

// 複数の制約
fun <T> process(item: T) where T : Comparable<T>, T : Serializable {
    // TはComparableかつSerializable
}
```

---

## 共変 (out) と反変 (in)

```kotlin
// 共変: out（読み取り専用、producerとして使用）
interface Producer<out T> {
    fun produce(): T
    // fun consume(item: T)  // エラー！outはパラメータ位置に使えない
}

// 反変: in（書き込み専用、consumerとして使用）
interface Consumer<in T> {
    fun consume(item: T)
    // fun produce(): T  // エラー！inは戻り値位置に使えない
}

// 実用例
open class Animal
class Dog : Animal()

val dogProducer: Producer<Dog> = object : Producer<Dog> {
    override fun produce(): Dog = Dog()
}
// outなのでProducer<Dog>をProducer<Animal>として使える
val animalProducer: Producer<Animal> = dogProducer

val animalConsumer: Consumer<Animal> = object : Consumer<Animal> {
    override fun consume(item: Animal) { /* 処理 */ }
}
// inなのでConsumer<Animal>をConsumer<Dog>として使える
val dogConsumer: Consumer<Dog> = animalConsumer
```

---

## スター投影

```kotlin
// 型パラメータが不明な場合
fun printAll(list: List<*>) {
    list.forEach { println(it) } // Any?として扱われる
}

// MutableListの場合
fun addToList(list: MutableList<*>) {
    // list.add("hello") // エラー！型が不明なので追加できない
    val item = list[0]   // Any?として取得はできる
}
```

---

## reified型パラメータ

```kotlin
// inline + reified で型情報を保持
inline fun <reified T> isInstanceOf(value: Any): Boolean {
    return value is T // 通常のジェネリクスではエラーになる
}

// Android実践: Intent起動
inline fun <reified T : Activity> Context.startActivity() {
    startActivity(Intent(this, T::class.java))
}

// 使用例
context.startActivity<SettingsActivity>()

// JSON パース
inline fun <reified T> Gson.fromJson(json: String): T {
    return fromJson(json, T::class.java)
}

val user = gson.fromJson<User>(jsonString)
```

---

## 実践パターン: Result型

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

// Repository
class UserRepository {
    suspend fun getUser(id: String): Result<User> {
        return try {
            val user = api.fetchUser(id)
            Result.Success(user)
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
}

// ViewModel
class UserViewModel : ViewModel() {
    private val _state = MutableStateFlow<Result<User>>(Result.Loading)
    val state: StateFlow<Result<User>> = _state.asStateFlow()
}

// Compose
@Composable
fun UserScreen(state: Result<User>) {
    when (state) {
        is Result.Loading -> CircularProgressIndicator()
        is Result.Success -> UserContent(state.data)
        is Result.Error -> ErrorMessage(state.exception.message)
    }
}
```

---

## 実践パターン: 型安全Builder

```kotlin
class ListBuilder<T> {
    private val items = mutableListOf<T>()

    fun add(item: T) = apply { items.add(item) }
    fun addAll(vararg items: T) = apply { this.items.addAll(items) }
    fun build(): List<T> = items.toList()
}

fun <T> buildList(block: ListBuilder<T>.() -> Unit): List<T> {
    return ListBuilder<T>().apply(block).build()
}

val users = buildList<String> {
    add("Alice")
    add("Bob")
    addAll("Charlie", "Dave")
}
```

---

## まとめ

- `<T>`で型パラメータ、`<T : Interface>`で型制約
- `out`（共変）: Producer側、`in`（反変）: Consumer側
- `*`（スター投影）: 型が不明な場合
- `reified`: inline関数内で型情報を保持
- `Result<out T>`: sealed classとの組み合わせが強力
- `Nothing`: すべての型のサブタイプ（Errorに使用）

---

8種類のAndroidアプリテンプレート（型安全設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [data classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-2026)
- [委譲パターンガイド](https://zenn.dev/myougatheaxo/articles/kotlin-delegation-2026)
- [Coroutines & Flowガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
