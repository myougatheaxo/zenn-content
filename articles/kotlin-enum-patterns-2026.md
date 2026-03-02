---
title: "Kotlin enumクラス実践ガイド — プロパティ/関数/シリアライズ"
emoji: "🏷️"
type: "tech"
topics: ["android", "kotlin", "enum", "basics"]
published: true
---

## この記事で学べること

Kotlinの**enumクラス**（プロパティ、関数、シリアライズ、Compose連携）を解説します。

---

## 基本のenum

```kotlin
enum class Priority {
    LOW, MEDIUM, HIGH, URGENT
}

val p = Priority.HIGH
println(p.name)     // HIGH
println(p.ordinal)  // 2

// 全値の取得
Priority.entries.forEach { println(it) }

// 文字列からの変換
val fromString = Priority.valueOf("HIGH") // Priority.HIGH
```

---

## プロパティ付きenum

```kotlin
enum class HttpStatus(val code: Int, val message: String) {
    OK(200, "OK"),
    NOT_FOUND(404, "Not Found"),
    INTERNAL_ERROR(500, "Internal Server Error"),
    UNAUTHORIZED(401, "Unauthorized");

    val isSuccess: Boolean get() = code in 200..299
    val isError: Boolean get() = code >= 400

    companion object {
        fun fromCode(code: Int): HttpStatus? =
            entries.find { it.code == code }
    }
}

// 使用
val status = HttpStatus.fromCode(404) // NOT_FOUND
println(status?.message)              // Not Found
println(status?.isError)              // true
```

---

## 関数付きenum（抽象メソッド）

```kotlin
enum class SortOrder {
    ASCENDING {
        override fun <T : Comparable<T>> sort(list: List<T>) = list.sorted()
    },
    DESCENDING {
        override fun <T : Comparable<T>> sort(list: List<T>) = list.sortedDescending()
    };

    abstract fun <T : Comparable<T>> sort(list: List<T>): List<T>
}

val numbers = listOf(3, 1, 4, 1, 5)
SortOrder.ASCENDING.sort(numbers)  // [1, 1, 3, 4, 5]
SortOrder.DESCENDING.sort(numbers) // [5, 4, 3, 1, 1]
```

---

## enumとwhen

```kotlin
enum class Screen {
    HOME, SEARCH, PROFILE, SETTINGS
}

fun getTitle(screen: Screen): String = when (screen) {
    Screen.HOME -> "ホーム"
    Screen.SEARCH -> "検索"
    Screen.PROFILE -> "プロフィール"
    Screen.SETTINGS -> "設定"
    // 網羅的チェック（追加忘れでコンパイルエラー）
}
```

---

## インターフェース実装

```kotlin
interface Displayable {
    val displayName: String
    val icon: String
}

enum class Category : Displayable {
    FOOD {
        override val displayName = "食品"
        override val icon = "🍔"
    },
    DRINK {
        override val displayName = "飲料"
        override val icon = "🥤"
    },
    SNACK {
        override val displayName = "お菓子"
        override val icon = "🍪"
    }
}
```

---

## Compose連携

```kotlin
enum class ThemeMode(val label: String) {
    LIGHT("ライト"),
    DARK("ダーク"),
    SYSTEM("システム設定");
}

@Composable
fun ThemeSelector(
    current: ThemeMode,
    onSelect: (ThemeMode) -> Unit
) {
    Column {
        ThemeMode.entries.forEach { mode ->
            Row(
                Modifier
                    .fillMaxWidth()
                    .clickable { onSelect(mode) }
                    .padding(16.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(
                    selected = mode == current,
                    onClick = { onSelect(mode) }
                )
                Spacer(Modifier.width(8.dp))
                Text(mode.label)
            }
        }
    }
}
```

---

## Roomとの連携

```kotlin
enum class TaskStatus {
    TODO, IN_PROGRESS, DONE
}

@Entity
data class Task(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val status: TaskStatus // Roomが自動で文字列変換
)

@Dao
interface TaskDao {
    @Query("SELECT * FROM task WHERE status = :status")
    fun getByStatus(status: TaskStatus): Flow<List<Task>>
}
```

---

## JSONシリアライズ

```kotlin
@Serializable
enum class Role {
    @SerialName("admin") ADMIN,
    @SerialName("user") USER,
    @SerialName("guest") GUEST
}

@Serializable
data class UserResponse(
    val name: String,
    val role: Role
)

// {"name":"Alice","role":"admin"}
val json = Json.encodeToString(UserResponse("Alice", Role.ADMIN))
```

---

## まとめ

- `enum class`でタイプセーフな定数定義
- プロパティ・関数・companion objectでロジック追加
- `when`で網羅的チェック（追加忘れ防止）
- `entries`で全値のリスト取得
- インターフェース実装で共通契約
- Room/Serialization連携でデータ永続化

---

8種類のAndroidアプリテンプレート（enumパターン活用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [sealed interfaceガイド](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-interface-2026)
- [data classガイド](https://zenn.dev/myougatheaxo/articles/kotlin-data-class-2026)
