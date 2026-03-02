---
title: "リコンポジション最適化ガイド — 不要な再描画を防ぐ"
emoji: "⚡"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

Composeの**リコンポジション最適化**（安定性、ラムダキャッシュ、derivedStateOf）を解説します。

---

## リコンポジションの仕組み

```kotlin
// ❌ 毎回リコンポーズされる
@Composable
fun UnstableExample(items: List<String>) { // Listは不安定
    items.forEach { Text(it) }
}

// ✅ 安定した型で不要なリコンポーズを防ぐ
@Immutable
data class StableItems(val items: List<String>)

@Composable
fun StableExample(stableItems: StableItems) {
    stableItems.items.forEach { Text(it) }
}
```

---

## @Stable / @Immutable

```kotlin
@Immutable
data class UserProfile(
    val id: String,
    val name: String,
    val avatarUrl: String
)

@Stable
class CounterState {
    var count by mutableIntStateOf(0)
        private set
    fun increment() { count++ }
}
```

---

## ラムダのキャッシュ

```kotlin
// ❌ 毎回新しいラムダが生成される
@Composable
fun BadList(items: List<Item>, viewModel: ItemViewModel) {
    LazyColumn {
        items(items) { item ->
            ItemRow(
                item = item,
                onClick = { viewModel.selectItem(item.id) } // 毎回新インスタンス
            )
        }
    }
}

// ✅ rememberでキャッシュ
@Composable
fun GoodList(items: List<Item>, viewModel: ItemViewModel) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            val onClick = remember(item.id) { { viewModel.selectItem(item.id) } }
            ItemRow(item = item, onClick = onClick)
        }
    }
}
```

---

## derivedStateOf

```kotlin
@Composable
fun OptimizedCounter() {
    var count by remember { mutableIntStateOf(0) }

    // ❌ countが変わるたびにリコンポーズ
    // val showWarning = count > 10

    // ✅ 結果が変わった時のみリコンポーズ
    val showWarning by remember { derivedStateOf { count > 10 } }

    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) { Text("+1") }
        if (showWarning) {
            Text("警告: 10を超えました", color = MaterialTheme.colorScheme.error)
        }
    }
}
```

---

## Composerレポートの活用

```kotlin
// build.gradle.kts
// コンパイラレポートでリコンポジション問題を検出
kotlinOptions {
    freeCompilerArgs += listOf(
        "-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=${layout.buildDirectory.get()}/compose_metrics",
        "-P", "plugin:androidx.compose.compiler.plugins.kotlin:metricsDestination=${layout.buildDirectory.get()}/compose_metrics"
    )
}

// レポート出力:
// build/compose_metrics/app_release-classes.txt
// → restartable / skippable / stable の情報
```

---

## まとめ

- `@Immutable`/`@Stable`でクラスの安定性を保証
- `key`パラメータでLazyListの再利用を最適化
- ラムダは`remember`でキャッシュ
- `derivedStateOf`で派生値の計算を最適化
- コンパイラレポートで安定性を確認
- `List`→`ImmutableList`（kotlinx-collections）で安定化

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
- [rememberパターン](https://zenn.dev/myougatheaxo/articles/android-compose-remember-patterns-2026)
- [Side Effectsガイド](https://zenn.dev/myougatheaxo/articles/android-compose-side-effects-2026)
