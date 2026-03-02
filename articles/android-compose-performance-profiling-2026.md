---
title: "Composeパフォーマンス計測ガイド — Layout Inspector/Profiler"
emoji: "📈"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

Composeの**パフォーマンス計測**（Layout Inspector、CPU Profiler、リコンポジション回数）を解説します。

---

## Layout Inspector

```kotlin
// Android StudioのLayout Inspectorで確認できる情報:
// - Composableツリー構造
// - リコンポジション回数（recomposition count）
// - スキップ回数（skip count）
// - 各Composableのパラメータ値

// Tools > Layout Inspector > 対象プロセス選択
// "Show Recomposition Counts" をONにする
```

---

## リコンポジション最適化チェック

```kotlin
// ❌ 不安定なパラメータ → 毎回リコンポーズ
@Composable
fun UnstableList(items: List<String>) { // List<String>は不安定
    items.forEach { Text(it) }
}

// ✅ 安定な型にラップ
@Immutable
data class StableList(val items: List<String>)

@Composable
fun StableListView(data: StableList) {
    data.items.forEach { Text(it) }
}

// ✅ key付きLazyColumn
LazyColumn {
    items(items, key = { it.id }) { item ->
        ItemRow(item)
    }
}
```

---

## コンパイラメトリクス

```kotlin
// build.gradle.kts
// Composeコンパイラの安定性レポート生成
kotlinOptions {
    freeCompilerArgs += listOf(
        "-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=" +
            "${layout.buildDirectory.get()}/compose_metrics",
        "-P", "plugin:androidx.compose.compiler.plugins.kotlin:metricsDestination=" +
            "${layout.buildDirectory.get()}/compose_metrics"
    )
}

// 出力ファイル:
// compose_metrics/app_release-classes.txt
//   → 各クラスの stable / unstable 情報
// compose_metrics/app_release-composables.txt
//   → 各Composableの restartable / skippable 情報
```

---

## CPU Profiler

```kotlin
// Android Studio > Profiler > CPU
// 1. "Record" を開始
// 2. アプリを操作
// 3. "Stop" で記録終了
// 4. フレームチャートで処理時間を確認

// Trace APIで特定区間を計測
import androidx.tracing.trace

fun processData(items: List<Item>): List<ProcessedItem> {
    return trace("processData") {
        items.map { item ->
            trace("processItem") {
                transform(item)
            }
        }
    }
}
```

---

## パフォーマンスTips

```kotlin
// 1. remember で計算結果をキャッシュ
val sorted = remember(items) { items.sortedBy { it.name } }

// 2. derivedStateOf で不要なリコンポーズ防止
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}

// 3. LazyColumn の key 指定
items(items, key = { it.id }) { item -> ItemRow(item) }

// 4. lambda のキャッシュ
val onClick = remember(item.id) { { viewModel.select(item.id) } }

// 5. Modifier の再利用
val cardModifier = remember {
    Modifier.fillMaxWidth().padding(16.dp).clip(RoundedCornerShape(12.dp))
}

// 6. graphicsLayer で描画最適化
Modifier.graphicsLayer { alpha = 0.99f } // 別レイヤーで描画
```

---

## まとめ

- Layout Inspectorでリコンポジション回数を確認
- コンパイラレポートで`stable`/`unstable`を検出
- `@Immutable`/`@Stable`で安定性を保証
- CPU Profilerでフレーム落ちの原因特定
- `remember`/`derivedStateOf`/`key`で最適化
- `graphicsLayer`で重い描画を別レイヤー化

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [リコンポジション最適化](https://zenn.dev/myougatheaxo/articles/android-compose-recomposition-debug-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
- [App Startup最適化](https://zenn.dev/myougatheaxo/articles/android-compose-app-startup-2026)
