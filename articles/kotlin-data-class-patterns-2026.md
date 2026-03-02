---
title: "Kotlin data class実践パターン — copy/destructuring/equals"
emoji: "📝"
type: "tech"
topics: ["android", "kotlin", "dataclass", "tips"]
published: true
---

## この記事で学べること

Kotlinの**data class**（copy、分解宣言、equals/hashCode、制限事項）を解説します。

---

## data classの基本

```kotlin
data class User(
    val id: String,
    val name: String,
    val email: String,
    val age: Int
)

// 自動生成される関数:
// - equals() / hashCode()
// - toString() → "User(id=1, name=Alice, ...)"
// - copy()
// - componentN()
```

---

## copy — 一部を変更したコピー

```kotlin
val user = User("1", "Alice", "alice@example.com", 25)

// 名前だけ変更
val updated = user.copy(name = "Alice Updated")

// 複数フィールド変更
val modified = user.copy(name = "Bob", age = 30)

// ViewModelでの活用
class UserViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state = _state.asStateFlow()

    fun updateName(name: String) {
        _state.update { it.copy(name = name) }
    }

    fun toggleLoading() {
        _state.update { it.copy(isLoading = !it.isLoading) }
    }
}

data class UiState(
    val name: String = "",
    val email: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)
```

---

## 分解宣言 (Destructuring)

```kotlin
val user = User("1", "Alice", "alice@example.com", 25)

// 分解宣言
val (id, name, email, age) = user

// mapでの活用
val map = mapOf("key1" to "value1", "key2" to "value2")
for ((key, value) in map) {
    println("$key = $value")
}

// Pairの分解
val (first, second) = Pair("Hello", 42)

// 関数からの複数戻り値
data class ValidationResult(val isValid: Boolean, val errors: List<String>)
val (isValid, errors) = validate(input)
```

---

## カスタムequals対象外フィールド

```kotlin
// data classのequals/hashCodeはコンストラクタのプロパティのみ対象
data class CacheEntry(
    val key: String,
    val value: String
) {
    // body内のプロパティはequals対象外
    var lastAccessTime: Long = System.currentTimeMillis()
    var hitCount: Int = 0
}

val entry1 = CacheEntry("k", "v")
val entry2 = CacheEntry("k", "v")
println(entry1 == entry2) // true（lastAccessTimeは無視）
```

---

## data classの制限と対策

```kotlin
// ❌ 継承できない（openにできない）
// data class Base(val x: Int)
// data class Child(val y: Int) : Base(1) // エラー

// ✅ interfaceは実装可能
interface Identifiable { val id: String }
data class Product(
    override val id: String,
    val name: String,
    val price: Int
) : Identifiable

// ✅ sealed classと組み合わせ
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val message: String) : Result()
    data object Loading : Result()
}
```

---

## まとめ

- `data class`はequals/hashCode/toString/copy/componentNを自動生成
- `copy()`で不変オブジェクトの一部変更（ViewModel状態管理に最適）
- 分解宣言でmap/Pair/関数の複数戻り値を簡潔に
- body内プロパティはequals対象外
- 継承不可だがinterface実装/sealed classとの組み合わせで拡張可能
- デフォルト引数で柔軟なインスタンス作成

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [enumパターン](https://zenn.dev/myougatheaxo/articles/kotlin-enum-patterns-2026)
- [ジェネリクスガイド](https://zenn.dev/myougatheaxo/articles/kotlin-generics-reified-2026)
