---
title: "Kotlin data class完全ガイド — 実践パターンとベストプラクティス"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "jetpackcompose", "初心者"]
published: true
---

## この記事で学べること

Kotlin の **data class** の基本から、Android開発での実践的な使い方まで解説します。

---

## 基本のdata class

```kotlin
data class User(
    val id: Int,
    val name: String,
    val email: String
)

// 自動生成されるもの
// - equals() / hashCode()
// - toString() → "User(id=1, name=太郎, email=...)"
// - copy()
// - componentN() → 分割代入
```

---

## copy（部分変更）

```kotlin
val user = User(1, "太郎", "taro@example.com")
val updated = user.copy(name = "花子")
// User(id=1, name=花子, email=taro@example.com)
```

**Composeでの活用**: State更新に最適。

```kotlin
data class UiState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

// ViewModelで
_uiState.update { it.copy(isLoading = true) }
```

---

## 分割代入

```kotlin
val (id, name, email) = user
println("$name ($email)")

// Mapのイテレーションでも
map.forEach { (key, value) ->
    println("$key: $value")
}
```

---

## APIレスポンスモデル

```kotlin
data class ApiResponse<T>(
    val data: T,
    val status: Int,
    val message: String
)

data class UserDto(
    @SerializedName("user_name")
    val userName: String,
    @SerializedName("created_at")
    val createdAt: String,
    val email: String
) {
    fun toUser(): User = User(
        name = userName,
        email = email,
        joinDate = createdAt
    )
}
```

---

## Room Entity

```kotlin
@Entity(tableName = "tasks")
data class TaskEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val title: String,
    val isCompleted: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)
```

---

## Composeの状態管理

```kotlin
data class FormState(
    val name: String = "",
    val email: String = "",
    val nameError: String? = null,
    val emailError: String? = null,
    val isValid: Boolean = false
)

class FormViewModel : ViewModel() {
    var formState by mutableStateOf(FormState())
        private set

    fun onNameChange(name: String) {
        formState = formState.copy(
            name = name,
            nameError = if (name.isBlank()) "名前を入力してください" else null,
            isValid = name.isNotBlank() && formState.email.contains("@")
        )
    }
}
```

---

## デフォルト値

```kotlin
data class Config(
    val theme: String = "light",
    val fontSize: Int = 14,
    val showNotifications: Boolean = true,
    val language: String = "ja"
)

// 必要な部分だけ指定
val config = Config(theme = "dark")
```

---

## data classの制約

```kotlin
// ❌ data classの継承はできない
// data class Admin(...) : User(...)

// ✅ sealed classと組み合わせる
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

// ✅ interfaceの実装は可能
interface Displayable {
    val displayName: String
}

data class Product(
    val id: Int,
    val name: String,
    val price: Int
) : Displayable {
    override val displayName: String get() = "$name (¥$price)"
}
```

---

## data class vs 通常のclass

| | data class | 通常のclass |
|--|-----------|------------|
| equals | プロパティ比較 | 参照比較 |
| toString | プロパティ表示 | ハッシュ値 |
| copy | あり | なし |
| componentN | あり | なし |
| 用途 | データ保持 | ロジック中心 |

---

## まとめ

- `data class` でequals/toString/copy/componentNが自動生成
- `copy()` で不変データの部分更新（State管理に最適）
- APIレスポンス、Room Entity、UI Stateのモデルとして活用
- `sealed class` と組み合わせて型安全なパターンマッチング
- デフォルト引数で柔軟な初期化

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Kotlin Sealed Class完全ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [Kotlinスコープ関数ガイド](https://zenn.dev/myougatheaxo/articles/kotlin-scope-functions-2026)
- [ViewModel完全ガイド](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
