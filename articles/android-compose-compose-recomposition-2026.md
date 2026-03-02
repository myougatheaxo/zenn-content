---
title: "再コンポーズ最適化完全ガイド — Stability/Skippable/ImmutableList/パフォーマンス"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**再コンポーズ最適化**（Stability、Skippable関数、ImmutableList、パフォーマンスチューニング）を解説します。

---

## Stabilityの基本

```kotlin
// ✅ Stable（自動スキップ可能）
data class StableUser(
    val id: Int,
    val name: String,
    val email: String
)

// ❌ Unstable（毎回再コンポーズ）
data class UnstableUser(
    val id: Int,
    val name: String,
    val tags: List<String>  // List<T>はUnstable
)

// ✅ 修正: ImmutableListを使用
@Immutable
data class FixedUser(
    val id: Int,
    val name: String,
    val tags: ImmutableList<String>  // kotlinx.collections.immutable
)
```

---

## Skippable関数の設計

```kotlin
// ❌ 毎回再コンポーズ（ラムダが毎回新規生成）
@Composable
fun BadExample(items: List<Item>) {
    LazyColumn {
        items(items) { item ->
            ItemCard(
                item = item,
                onClick = { doSomething(item) }  // 新しいラムダ
            )
        }
    }
}

// ✅ rememberで安定化
@Composable
fun GoodExample(items: List<Item>, viewModel: ItemViewModel) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            val onClick = remember(item.id) { { viewModel.onItemClick(item.id) } }
            ItemCard(item = item, onClick = onClick)
        }
    }
}

// ✅ Stableなコンポーネント
@Composable
fun ItemCard(
    item: StableItem,        // Stableなデータクラス
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(modifier = modifier.clickable(onClick = onClick)) {
        Text(item.title)
    }
}
```

---

## パフォーマンス計測

```kotlin
// Composition Tracing
@Composable
fun TracedList(items: List<Item>) {
    trace("TracedList") {
        LazyColumn {
            items(items, key = { it.id }) { item ->
                trace("ItemRow-${item.id}") {
                    ItemRow(item = item)
                }
            }
        }
    }
}

// リコンポーズカウンター（デバッグ用）
@Composable
fun RecompositionCounter(label: String) {
    val count = remember { mutableIntStateOf(0) }
    SideEffect { count.intValue++ }

    if (BuildConfig.DEBUG) {
        Text("$label: ${count.intValue}回", color = Color.Red, fontSize = 10.sp)
    }
}
```

---

## まとめ

| 状態 | 結果 |
|------|------|
| Stable + 同じ引数 | スキップ |
| Unstable | 毎回再コンポーズ |
| ラムダ新規生成 | 毎回再コンポーズ |
| `key` 付き | 効率的差分更新 |

- `@Immutable`/`@Stable`で型を安定化
- `ImmutableList`でコレクションを安定化
- `key`でLazyListの差分更新を最適化
- Compiler Reportで安定性を確認

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Compiler Report](https://zenn.dev/myougatheaxo/articles/android-compose-compose-compiler-report-2026)
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-baseline-profile-2026)
