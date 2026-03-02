---
title: "SegmentedButton実装ガイド — Composeでセグメント切り替えUI"
emoji: "🔘"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**SegmentedButton**（iOS風セグメントコントロール）をComposeで実装する方法を解説します。

---

## 単一選択SegmentedButton

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SingleSelectSegment() {
    val options = listOf("日", "週", "月")
    var selectedIndex by remember { mutableIntStateOf(0) }

    SingleChoiceSegmentedButtonRow(Modifier.fillMaxWidth()) {
        options.forEachIndexed { index, label ->
            SegmentedButton(
                shape = SegmentedButtonDefaults.itemShape(
                    index = index,
                    count = options.size
                ),
                onClick = { selectedIndex = index },
                selected = selectedIndex == index
            ) {
                Text(label)
            }
        }
    }
}
```

---

## 複数選択SegmentedButton

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MultiSelectSegment() {
    val options = listOf("太字" to Icons.Default.FormatBold,
                         "斜体" to Icons.Default.FormatItalic,
                         "下線" to Icons.Default.FormatUnderlined)
    var selectedOptions by remember { mutableStateOf(setOf<Int>()) }

    MultiChoiceSegmentedButtonRow(Modifier.fillMaxWidth()) {
        options.forEachIndexed { index, (label, icon) ->
            SegmentedButton(
                shape = SegmentedButtonDefaults.itemShape(
                    index = index,
                    count = options.size
                ),
                checked = index in selectedOptions,
                onCheckedChange = {
                    selectedOptions = if (index in selectedOptions)
                        selectedOptions - index
                    else
                        selectedOptions + index
                },
                icon = { Icon(icon, label, Modifier.size(18.dp)) }
            ) {
                Text(label)
            }
        }
    }
}
```

---

## アイコン付きSegmentedButton

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ViewModeSegment() {
    val modes = listOf(
        "リスト" to Icons.AutoMirrored.Filled.ViewList,
        "グリッド" to Icons.Default.GridView,
        "マップ" to Icons.Default.Map
    )
    var selectedMode by remember { mutableIntStateOf(0) }

    SingleChoiceSegmentedButtonRow {
        modes.forEachIndexed { index, (label, icon) ->
            SegmentedButton(
                shape = SegmentedButtonDefaults.itemShape(index, modes.size),
                onClick = { selectedMode = index },
                selected = selectedMode == index,
                icon = { Icon(icon, null, Modifier.size(18.dp)) }
            ) {
                Text(label)
            }
        }
    }
}
```

---

## コンテンツ切り替えと連携

```kotlin
@Composable
fun SegmentedContent() {
    var selectedTab by remember { mutableIntStateOf(0) }
    val tabs = listOf("すべて", "未完了", "完了")

    Column(Modifier.padding(16.dp)) {
        SingleChoiceSegmentedButtonRow(Modifier.fillMaxWidth()) {
            tabs.forEachIndexed { index, label ->
                SegmentedButton(
                    shape = SegmentedButtonDefaults.itemShape(index, tabs.size),
                    onClick = { selectedTab = index },
                    selected = selectedTab == index
                ) { Text(label) }
            }
        }

        Spacer(Modifier.height(16.dp))

        Crossfade(targetState = selectedTab) { tab ->
            when (tab) {
                0 -> AllTasksList()
                1 -> PendingTasksList()
                2 -> CompletedTasksList()
            }
        }
    }
}
```

---

## まとめ

- `SingleChoiceSegmentedButtonRow`で単一選択
- `MultiChoiceSegmentedButtonRow`で複数選択
- `SegmentedButtonDefaults.itemShape`で角丸制御
- `icon`パラメータでアイコン付き
- `Crossfade`でコンテンツ切り替えアニメーション

---

8種類のAndroidアプリテンプレート（UIコンポーネント設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [TabRow + HorizontalPager](https://zenn.dev/myougatheaxo/articles/android-compose-tab-pager-2026)
- [FilterChip実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-chip-filter-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
