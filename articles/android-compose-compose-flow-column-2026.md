---
title: "FlowColumn完全ガイド — 縦方向折り返し/タグクラウド/カテゴリ配置"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "layout"]
published: true
---

## この記事で学べること

**FlowColumn**（縦方向の折り返しレイアウト、maxItemsInEachColumn、動的配置）を解説します。

---

## FlowColumn基本

```kotlin
@OptIn(ExperimentalLayoutApi::class)
@Composable
fun FlowColumnBasic() {
    FlowColumn(
        modifier = Modifier.fillMaxWidth().height(300.dp).padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        maxItemsInEachColumn = 5
    ) {
        val items = listOf("Kotlin", "Java", "Swift", "Dart", "Python", "Rust", "Go", "TypeScript", "C++", "Ruby")
        items.forEach { lang ->
            AssistChip(onClick = {}, label = { Text(lang) })
        }
    }
}
```

---

## カテゴリ一覧

```kotlin
@OptIn(ExperimentalLayoutApi::class)
@Composable
fun CategoryGrid() {
    val categories = mapOf(
        "フロントエンド" to listOf("React", "Vue", "Angular", "Svelte"),
        "バックエンド" to listOf("Spring", "Django", "Express", "FastAPI"),
        "モバイル" to listOf("Android", "iOS", "Flutter", "React Native")
    )

    FlowColumn(
        Modifier.fillMaxWidth().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        categories.forEach { (category, items) ->
            Card(Modifier.width(160.dp)) {
                Column(Modifier.padding(12.dp)) {
                    Text(category, style = MaterialTheme.typography.titleSmall)
                    Spacer(Modifier.height(4.dp))
                    items.forEach { item ->
                        Text("• $item", style = MaterialTheme.typography.bodySmall)
                    }
                }
            }
        }
    }
}
```

---

## 統計ダッシュボード

```kotlin
@OptIn(ExperimentalLayoutApi::class)
@Composable
fun StatsDashboard() {
    val stats = listOf(
        "ダウンロード" to "1,234", "レビュー" to "56",
        "評価" to "4.5", "アクティブ" to "890",
        "収益" to "¥12,300", "更新" to "3日前"
    )

    FlowColumn(
        Modifier.fillMaxWidth().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        maxItemsInEachColumn = 3
    ) {
        stats.forEach { (label, value) ->
            Card(Modifier.width(IntrinsicSize.Max)) {
                Column(Modifier.padding(12.dp), horizontalAlignment = Alignment.CenterHorizontally) {
                    Text(value, style = MaterialTheme.typography.headlineSmall, fontWeight = FontWeight.Bold)
                    Text(label, style = MaterialTheme.typography.bodySmall, color = Color.Gray)
                }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FlowColumn` | 縦方向折り返し |
| `maxItemsInEachColumn` | 列あたり最大数 |
| `verticalArrangement` | 縦方向間隔 |
| `horizontalArrangement` | 列間の間隔 |

- `FlowColumn`で縦方向に折り返すレイアウト
- `maxItemsInEachColumn`で列の最大アイテム数制限
- `FlowRow`と使い分けてレスポンシブUI
- ダッシュボード/カテゴリ一覧に活用

---

8種類のAndroidアプリテンプレート（レイアウト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [FlowRow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-flow-row-2026)
- [LazyStaggeredGrid](https://zenn.dev/myougatheaxo/articles/android-compose-compose-lazy-staggered-2026)
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
