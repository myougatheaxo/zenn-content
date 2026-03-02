---
title: "Kotlinジェネリクス完全ガイド — 型パラメータ/共変/反変/reified"
emoji: "🔷"
type: "tech"
topics: ["android", "kotlin", "generics", "programming"]
published: true
---

## この記事で学べること

**Kotlinジェネリクス**（型パラメータ、共変out/反変in、型制約、reified、スター投影）を解説します。

---

## 基本ジェネリクス

```kotlin
// ジェネリッククラス
class Result<T>(val data: T?, val error: String? = null) {
    val isSuccess: Boolean get() = data != null

    fun getOrDefault(default: T): T = data ?: default

    fun <R> map(transform: (T) -> R): Result<R> =
        if (data != null) Result(transform(data))
        else Result(null, error)
}

// 使用
val result: Result<String> = Result("Hello")
val length: Result<Int> = result.map { it.length }

// ジェネリック関数
fun <T : Comparable<T>> List<T>.maxOrNull(): T? =
    if (isEmpty()) null else reduce { a, b -> if (a > b) a else b }
```

---

## 共変（out）と反変（in）

```kotlin
// 共変: out = 出力のみ（Producer）
interface Source<out T> {
    fun next(): T    // OK: Tを返す
    // fun add(t: T) // NG: Tを引数にできない
}

// List<out E> は共変
val strings: List<String> = listOf("a", "b")
val objects: List<Any> = strings // OK: String は Any のサブタイプ

// 反変: in = 入力のみ（Consumer）
interface Sink<in T> {
    fun add(t: T)    // OK: Tを引数にする
    // fun get(): T  // NG: Tを返せない
}

// Comparable<in T> は反変
val comparator: Comparable<Number> = object : Comparable<Number> {
    override fun compareTo(other: Number) = 0
}
val intComparable: Comparable<Int> = comparator // OK
```

---

## 型制約

```kotlin
// 上限境界
fun <T : Comparable<T>> sort(list: List<T>): List<T> =
    list.sorted()

// 複数の型制約（where）
fun <T> copyIfValid(item: T): T where T : Comparable<T>, T : Cloneable {
    return item
}

// 型制約付きクラス
class Repository<T : Entity>(private val dao: Dao<T>) {
    suspend fun getAll(): List<T> = dao.getAll()
    suspend fun insert(item: T) = dao.insert(item)
}

interface Entity {
    val id: String
}
```

---

## reified型パラメータ

```kotlin
// inline + reified で型情報を実行時に保持
inline fun <reified T> String.parseJson(): T {
    val gson = Gson()
    return gson.fromJson(this, T::class.java)
}

// 使用
val user = jsonString.parseJson<User>()

// Intentのextra取得
inline fun <reified T : Activity> Context.startActivity(
    vararg extras: Pair<String, Any>
) {
    val intent = Intent(this, T::class.java).apply {
        extras.forEach { (key, value) ->
            when (value) {
                is String -> putExtra(key, value)
                is Int -> putExtra(key, value)
                is Boolean -> putExtra(key, value)
            }
        }
    }
    startActivity(intent)
}

// 使用
context.startActivity<DetailActivity>("id" to 42)

// 型チェック
inline fun <reified T> List<*>.filterIsInstanceOf(): List<T> =
    filterIsInstance<T>()
```

---

## スター投影（*）

```kotlin
// 型引数が不明な場合
fun printAll(list: List<*>) {
    list.forEach { println(it) } // Any? として扱う
}

// MutableList<*> は読み取りのみ
fun readOnly(list: MutableList<*>) {
    val item = list[0]     // OK: Any? として取得
    // list.add("x")       // NG: 型が不明なので追加不可
}
```

---

## 実践パターン

```kotlin
// 型安全なビルダー
class RequestBuilder<T : Any> {
    private var url: String = ""
    private var parser: ((String) -> T)? = null

    fun url(url: String) = apply { this.url = url }
    fun parser(parser: (String) -> T) = apply { this.parser = parser }

    suspend fun execute(): Result<T> {
        val response = httpClient.get(url)
        return try {
            Result(parser!!(response))
        } catch (e: Exception) {
            Result(null, e.message)
        }
    }
}

inline fun <reified T : Any> request(
    block: RequestBuilder<T>.() -> Unit
): RequestBuilder<T> = RequestBuilder<T>().apply(block)

// sealed class + ジェネリクス
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String) : ApiResult<Nothing>()
    data object Loading : ApiResult<Nothing>()
}
```

---

## まとめ

| 概念 | キーワード | 用途 |
|------|-----------|------|
| 共変 | `out` | 出力のみ（Producer） |
| 反変 | `in` | 入力のみ（Consumer） |
| 上限境界 | `T : Type` | 型制約 |
| reified | `inline reified` | 実行時型情報 |
| スター投影 | `*` | 型引数不明 |

- `out`はList, Flow等の読み取り専用に
- `in`はComparator等の消費型に
- `reified`でGson/Intent等の型推論
- `Nothing`は全ての型のサブタイプ

---

8種類のAndroidアプリテンプレート（型安全設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed Result パターン](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-result-2026)
- [高階関数](https://zenn.dev/myougatheaxo/articles/kotlin-higher-order-functions-2026)
- [プロパティ委譲](https://zenn.dev/myougatheaxo/articles/kotlin-property-delegate-2026)
