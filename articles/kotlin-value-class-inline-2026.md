---
title: "Kotlin value class完全ガイド — 型安全/ゼロコスト抽象化/実践パターン"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "programming"]
published: true
---

## この記事で学べること

**value class**（インライン化、型安全、ドメインモデリング、Compose連携、制約事項）を解説します。

---

## value classとは

```kotlin
// 実行時にラップが消え、内部の値だけになる（ゼロコスト）
@JvmInline
value class UserId(val value: String)

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email: $value" }
    }
}

@JvmInline
value class Price(val yen: Int) {
    init {
        require(yen >= 0) { "Price must be non-negative" }
    }
    fun withTax(rate: Double = 0.1): Int = (yen * (1 + rate)).toInt()
}
```

---

## 型安全の威力

```kotlin
// ❌ String同士の取り違えが起きる
fun getUser(userId: String, email: String): User

// ✅ value classで型が異なるため取り違え不可
fun getUser(userId: UserId, email: Email): User

// コンパイルエラー！
getUser(Email("test@example.com"), UserId("user-123"))
```

---

## 実践パターン

```kotlin
@JvmInline
value class Milliseconds(val value: Long) {
    fun toSeconds(): Double = value / 1000.0
}

@JvmInline
value class Percentage(val value: Int) {
    init { require(value in 0..100) }
    fun toFloat(): Float = value / 100f
}

@JvmInline
value class Rgb(val value: Int) {
    val red: Int get() = (value shr 16) and 0xFF
    val green: Int get() = (value shr 8) and 0xFF
    val blue: Int get() = value and 0xFF
}
```

---

## Repository/ViewModelでの使用

```kotlin
class UserRepository @Inject constructor(private val api: UserApi) {
    suspend fun getUser(id: UserId): User = api.getUser(id.value)
    suspend fun getUserByEmail(email: Email): User = api.getUserByEmail(email.value)
}

@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val userId = UserId(savedStateHandle.get<String>("userId")!!)

    val user: StateFlow<User?> = flow { emit(repository.getUser(userId)) }
        .stateIn(viewModelScope, SharingStarted.Lazily, null)
}
```

---

## 制約事項

```kotlin
// ❌ value classでは使えないもの
// - 複数のプロパティ（コンストラクタにはvalが1つだけ）
// - identity（===比較は常にfalse）
// - ジェネリクスでBoxing発生

// Boxing発生するケース
val userId: UserId = UserId("123")     // ✅ インライン化
val nullable: UserId? = UserId("123")  // ❌ Boxing発生
val list: List<UserId> = listOf(...)   // ❌ Boxing発生
```

---

## まとめ

| 特徴 | 説明 |
|------|------|
| ゼロコスト | 実行時にラップ解除 |
| 型安全 | 同じ型の引数取り違え防止 |
| バリデーション | `init`で制約を強制 |
| メソッド | 変換・計算ロジック追加可能 |
| 制約 | プロパティ1つ、nullable/Genericでboxing |

- `@JvmInline value class`で型安全かつゼロコスト
- `init`ブロックでバリデーション
- ID、Email、金額など値オブジェクトに最適
- nullable/ジェネリクスではBoxing発生に注意

---

8種類のAndroidアプリテンプレート（Kotlin best practices適用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [data classパターン](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-patterns-2026)
- [sealed class](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [ジェネリクス](https://zenn.dev/myougatheaxo/articles/kotlin-generics-variance-2026)
