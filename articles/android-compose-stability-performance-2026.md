---
title: "Compose安定性と再コンポジション最適化 — @Stable/@Immutable/skippable"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Compose安定性**（@Stable、@Immutable、スキップ可能判定、Compose Compiler Report、パフォーマンス最適化）を解説します。

---

## 安定性とは

Composeは引数が変わっていなければ再コンポジションをスキップします。スキップできるかどうかは引数の「安定性」で決まります。

```kotlin
// ✅ 安定（自動判定）: プリミティブ、String、関数型
@Composable
fun Greeting(name: String) { // String = 安定 → スキップ可能
    Text("Hello, $name")
}

// ❌ 不安定: List, Mapなどのコレクション
@Composable
fun UserList(users: List<User>) { // List = 不安定 → 毎回再コンポジション
    LazyColumn {
        items(users) { UserItem(it) }
    }
}
```

---

## @Immutable

```kotlin
@Immutable
data class User(
    val id: String,
    val name: String,
    val email: String
)

// すべてのプロパティがvalかつ安定型なら@Immutableを付ける
// Composeはこのクラスのインスタンスが変更されないことを保証として扱う
```

---

## @Stable

```kotlin
@Stable
class CounterState {
    var count by mutableIntStateOf(0)
        private set

    fun increment() { count++ }
}

// @Stable = 「同じインスタンスに対してequals()が同じ結果を返す限りスキップ可能」
// MutableStateを含むクラスに使用
```

---

## コレクションの安定化

```kotlin
// 方法1: kotlinx.collections.immutable
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.toImmutableList

@Composable
fun UserList(users: ImmutableList<User>) { // ✅ 安定
    LazyColumn {
        items(users.size) { index -> UserItem(users[index]) }
    }
}

// 呼び出し側
val users = usersList.toImmutableList()

// 方法2: ラッパークラス
@Immutable
data class UserListWrapper(val users: List<User>)
```

---

## Compose Compiler Report

```kotlin
// build.gradle.kts
android {
    composeCompiler {
        reportsDestination = layout.buildDirectory.dir("compose_reports")
        metricsDestination = layout.buildDirectory.dir("compose_metrics")
    }
}
```

```
// レポート出力例
// app_release-composables.txt
restartable skippable scheme("[androidx.compose.ui.UnitComposable]") fun Greeting(
  stable name: String
)

restartable scheme("[androidx.compose.ui.UnitComposable]") fun UserList(
  unstable users: List<User>  // ← 不安定！スキップ不可
)
```

---

## derivedStateOf

```kotlin
@Composable
fun FilteredList(items: List<Item>, query: String) {
    // ❌ 毎回フィルタリング
    val filtered = items.filter { it.name.contains(query) }

    // ✅ 結果が変わった時だけ再計算
    val filtered by remember(items, query) {
        derivedStateOf { items.filter { it.name.contains(query) } }
    }

    LazyColumn {
        items(filtered) { item -> ItemRow(item) }
    }
}
```

---

## key による安定化

```kotlin
@Composable
fun ItemList(items: List<Item>) {
    LazyColumn {
        items(
            items = items,
            key = { it.id } // ✅ keyで同一性を保証 → 再利用
        ) { item ->
            ItemRow(item)
        }
    }
}
```

---

## remember + lambda安定化

```kotlin
// ❌ 毎回新しいlambdaインスタンス
@Composable
fun Parent(viewModel: MyViewModel) {
    Child(onClick = { viewModel.doSomething() })
}

// ✅ remember でlambdaを安定化
@Composable
fun Parent(viewModel: MyViewModel) {
    val onClick = remember(viewModel) { { viewModel.doSomething() } }
    Child(onClick = onClick)
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| `@Immutable` | 不変データクラス |
| `@Stable` | MutableState含むクラス |
| `ImmutableList` | コレクション安定化 |
| `derivedStateOf` | 派生状態の最適化 |
| `key` | LazyListアイテム同一性 |
| Compiler Report | 不安定箇所の発見 |

- Compose Compiler Reportで不安定な引数を発見
- `@Immutable`/`@Stable`で安定性を明示
- `kotlinx.collections.immutable`でコレクション安定化
- `derivedStateOf`で不要な再計算を防止

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [再コンポジションデバッグ](https://zenn.dev/myougatheaxo/articles/android-compose-recomposition-debug-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-baseline-profile-2026)
