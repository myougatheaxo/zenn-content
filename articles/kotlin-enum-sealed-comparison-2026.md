---
title: "enum vs sealed class vs sealed interface — 使い分け完全ガイド"
emoji: "🔍"
type: "tech"
topics: ["android", "kotlin", "enum", "sealed"]
published: true
---

## この記事で学べること

Kotlinの**enum/sealed class/sealed interface**の使い分け（判断基準、パターン、実践例）を解説します。

---

## 判断基準

```
固定の定数セット（値を持たない or 固定値のみ）
  → enum class

型ごとに異なるデータを持つ（可変な型階層）
  → sealed class / sealed interface

マルチモジュールで共有
  → sealed interface（複数実装可能）
```

---

## enum class

```kotlin
// 固定の定数セット
enum class Priority(val level: Int) {
    LOW(0),
    MEDIUM(1),
    HIGH(2);

    fun label(): String = when (this) {
        LOW -> "低"
        MEDIUM -> "中"
        HIGH -> "高"
    }
}

// 使用
val priority = Priority.HIGH
println(priority.level) // 2
println(priority.label()) // "高"

// 文字列から変換
val fromString = Priority.valueOf("HIGH")
val all = Priority.entries // [LOW, MEDIUM, HIGH]

// Compose Dropdown
@Composable
fun PriorityDropdown(selected: Priority, onSelect: (Priority) -> Unit) {
    var expanded by remember { mutableStateOf(false) }

    ExposedDropdownMenuBox(expanded = expanded, onExpandedChange = { expanded = it }) {
        OutlinedTextField(
            value = selected.label(),
            onValueChange = {},
            readOnly = true,
            modifier = Modifier.menuAnchor()
        )
        ExposedDropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
            Priority.entries.forEach { priority ->
                DropdownMenuItem(
                    text = { Text(priority.label()) },
                    onClick = { onSelect(priority); expanded = false }
                )
            }
        }
    }
}
```

---

## sealed class

```kotlin
// 型ごとに異なるデータ
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String, val cause: Throwable? = null) : UiState<Nothing>()
}

// when で網羅性チェック
fun <T> handleState(state: UiState<T>) {
    when (state) {
        UiState.Loading -> showLoading()
        is UiState.Success -> showData(state.data) // data にアクセス可能
        is UiState.Error -> showError(state.message, state.cause)
    } // 全パターン網羅 → elseブランチ不要
}

// ナビゲーションイベント
sealed class NavigationEvent {
    data class NavigateTo(val route: String) : NavigationEvent()
    data class NavigateToWithArgs(val route: String, val args: Map<String, Any>) : NavigationEvent()
    data object GoBack : NavigationEvent()
    data class GoBackWithResult(val key: String, val value: Any) : NavigationEvent()
}
```

---

## sealed interface

```kotlin
// 複数のsealed interfaceを実装可能
sealed interface Drawable {
    fun draw(canvas: Canvas)
}

sealed interface Clickable {
    fun onClick()
}

// 両方を実装
data class Button(val text: String) : Drawable, Clickable {
    override fun draw(canvas: Canvas) { /* ... */ }
    override fun onClick() { /* ... */ }
}

// sealed interfaceは状態を持たない場合に最適
sealed interface Permission {
    data object Granted : Permission
    data object Denied : Permission
    data class DeniedPermanently(val permission: String) : Permission
}

// マルチモジュール向け
// core-domain に定義
sealed interface DomainError {
    data class NetworkError(val code: Int) : DomainError
    data class DatabaseError(val message: String) : DomainError
    data object UnknownError : DomainError
}
```

---

## 比較表

```kotlin
// enum class
// ✅ 固定の定数セット
// ✅ entries, valueOf(), ordinal
// ✅ name プロパティ
// ❌ 型ごとに異なるデータ不可
// ❌ 状態のインスタンスを複数作れない

// sealed class
// ✅ 型ごとに異なるデータ
// ✅ when の網羅性チェック
// ✅ インスタンス生成可能
// ❌ 単一継承のみ

// sealed interface
// ✅ 型ごとに異なるデータ
// ✅ when の網羅性チェック
// ✅ 複数実装可能
// ✅ マルチモジュール向き
// ❌ コンストラクタのデフォルト値なし
```

---

## 実践: enum + sealed の組み合わせ

```kotlin
enum class TaskFilter { ALL, ACTIVE, COMPLETED }

sealed interface TaskAction {
    data class Add(val title: String) : TaskAction
    data class Toggle(val id: String) : TaskAction
    data class Delete(val id: String) : TaskAction
    data class SetFilter(val filter: TaskFilter) : TaskAction // enum を sealed 内で使用
}

@HiltViewModel
class TaskViewModel @Inject constructor() : ViewModel() {
    fun dispatch(action: TaskAction) {
        when (action) {
            is TaskAction.Add -> addTask(action.title)
            is TaskAction.Toggle -> toggleTask(action.id)
            is TaskAction.Delete -> deleteTask(action.id)
            is TaskAction.SetFilter -> setFilter(action.filter)
        }
    }
}
```

---

## まとめ

| 特徴 | enum | sealed class | sealed interface |
|------|------|-------------|-----------------|
| 固定定数 | ✅ | △ | △ |
| 異なるデータ | ❌ | ✅ | ✅ |
| 多重継承 | ❌ | ❌ | ✅ |
| when網羅性 | ✅ | ✅ | ✅ |
| entries取得 | ✅ | ❌ | ❌ |

- **enum**: 固定選択肢（Priority、Filter、Status）
- **sealed class**: UIState、Result、Event
- **sealed interface**: マルチモジュール、複数trait実装
- 組み合わせ: enum（値）をsealed（アクション）内で使う

---

8種類のAndroidアプリテンプレート（Kotlinベストプラクティス適用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [sealed class](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-class-2026)
- [Result型パターン](https://zenn.dev/myougatheaxo/articles/kotlin-sealed-result-2026)
- [enumパターン](https://zenn.dev/myougatheaxo/articles/kotlin-enum-patterns-2026)
