---
title: "Kotlinジェネリクス完全ガイド — in/out/reified/where"
emoji: "🧬"
type: "tech"
topics: ["android", "kotlin", "generics", "tips"]
published: true
---

## この記事で学べること

Kotlinの**ジェネリクス**（型パラメータ、変性、reified、型制約）を解説します。

---

## 基本のジェネリクス

```kotlin
// ジェネリッククラス
class Result<T>(val data: T?, val error: String? = null) {
    val isSuccess: Boolean get() = data != null
}

// ジェネリック関数
fun <T> List<T>.secondOrNull(): T? = if (size >= 2) this[1] else null

// 使用
val result = Result(data = "Hello")
val second = listOf(1, 2, 3).secondOrNull() // 2
```

---

## 変性: out (共変) / in (反変)

```kotlin
// out: 生産者（読み取り専用）
interface Source<out T> {
    fun next(): T // ✅ 戻り値に使える
    // fun add(item: T) // ❌ 引数には使えない
}

// in: 消費者（書き込み専用）
interface Sink<in T> {
    fun put(item: T) // ✅ 引数に使える
    // fun get(): T // ❌ 戻り値には使えない
}

// outの効果: サブタイプ関係を保持
val stringSource: Source<String> = createStringSource()
val anySource: Source<Any> = stringSource // ✅ OK（outなので）

// inの効果: サブタイプ関係が逆転
val anySink: Sink<Any> = createAnySink()
val stringSink: Sink<String> = anySink // ✅ OK（inなので）
```

---

## reified型パラメータ

```kotlin
// ❌ 型情報はランタイムで消える（型消去）
// fun <T> isType(value: Any): Boolean = value is T // コンパイルエラー

// ✅ inline + reified で型情報を保持
inline fun <reified T> isType(value: Any): Boolean = value is T

inline fun <reified T> List<Any>.filterByType(): List<T> =
    filterIsInstance<T>()

// Android: IntentのstartActivity
inline fun <reified T : Activity> Context.startActivity() {
    startActivity(Intent(this, T::class.java))
}

// 使用
context.startActivity<DetailActivity>()

// Gson等のデシリアライズ
inline fun <reified T> Gson.fromJsonTyped(json: String): T =
    fromJson(json, T::class.java)
```

---

## 型制約: where

```kotlin
// 単一の上限境界
fun <T : Comparable<T>> List<T>.customSort(): List<T> = sorted()

// 複数の型制約
fun <T> ensureSerializable(value: T): T
    where T : Serializable, T : Comparable<T> {
    return value
}

// 実用例: ViewModelでRepository制約
class ListViewModel<T>(
    private val repository: Repository<T>
) : ViewModel() where T : Identifiable, T : Serializable {
    // Tは Identifiable かつ Serializable
}
```

---

## スター投影 (*)

```kotlin
// 型パラメータが不明な場合
fun printAll(list: List<*>) {
    list.forEach { println(it) } // Any?として扱われる
}

// MutableListでは読み取りのみ
fun readOnly(list: MutableList<*>) {
    val item = list[0] // ✅ Any?として読める
    // list.add("x") // ❌ 書き込み不可
}
```

---

## まとめ

| 概念 | 構文 | 用途 |
|------|------|------|
| ジェネリクス | `<T>` | 型をパラメータ化 |
| 共変 | `<out T>` | 読み取り専用（生産者） |
| 反変 | `<in T>` | 書き込み専用（消費者） |
| reified | `inline <reified T>` | ランタイム型情報保持 |
| 型制約 | `where T : A, T : B` | 複数境界 |
| スター投影 | `<*>` | 型不明時 |

---

8種類のAndroidアプリテンプレート（型安全設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [関数型パターン](https://zenn.dev/myougatheaxo/articles/kotlin-functional-patterns-2026)
- [インターフェースパターン](https://zenn.dev/myougatheaxo/articles/kotlin-interface-patterns-2026)
