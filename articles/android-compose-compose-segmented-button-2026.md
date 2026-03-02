---
title: "SegmentedButton完全ガイド — SingleChoice/MultiChoice/カスタムスタイル"
emoji: "🔘"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**SegmentedButton**（SingleChoiceSegmentedButtonRow、MultiChoiceSegmentedButtonRow）を解説します。

---

## SingleChoice

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SingleChoiceSegmented() {
    var selectedIndex by remember { mutableIntStateOf(0) }
    val options = listOf("日", "週", "月")

    SingleChoiceSegmentedButtonRow(Modifier.fillMaxWidth()) {
        options.forEachIndexed { index, label ->
            SegmentedButton(
                selected = selectedIndex == index,
                onClick = { selectedIndex = index },
                shape = SegmentedButtonDefaults.itemShape(index, options.size)
            ) {
                Text(label)
            }
        }
    }
}
```

---

## MultiChoice

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MultiChoiceSegmented() {
    val options = listOf("朝", "昼", "夜")
    var selectedOptions by remember { mutableStateOf(setOf<Int>()) }

    MultiChoiceSegmentedButtonRow(Modifier.fillMaxWidth()) {
        options.forEachIndexed { index, label ->
            SegmentedButton(
                checked = index in selectedOptions,
                onCheckedChange = {
                    selectedOptions = if (index in selectedOptions)
                        selectedOptions - index else selectedOptions + index
                },
                shape = SegmentedButtonDefaults.itemShape(index, options.size),
                icon = {
                    SegmentedButtonDefaults.Icon(active = index in selectedOptions)
                }
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
fun IconSegmentedButton() {
    var selected by remember { mutableIntStateOf(0) }
    val options = listOf(
        Icons.Default.ViewList to "リスト",
        Icons.Default.GridView to "グリッド",
        Icons.Default.ViewModule to "モジュール"
    )

    SingleChoiceSegmentedButtonRow {
        options.forEachIndexed { index, (icon, label) ->
            SegmentedButton(
                selected = selected == index,
                onClick = { selected = index },
                shape = SegmentedButtonDefaults.itemShape(index, options.size),
                icon = { Icon(icon, label, Modifier.size(18.dp)) }
            ) {
                Text(label)
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SingleChoiceSegmentedButtonRow` | 単一選択 |
| `MultiChoiceSegmentedButtonRow` | 複数選択 |
| `SegmentedButton` | 個別ボタン |
| `itemShape` | ボタン形状 |

- `SingleChoice`で排他的な選択（表示モード等）
- `MultiChoice`で複数選択（フィルター等）
- `SegmentedButtonDefaults.Icon`でチェックアイコン
- `itemShape`で端のボタンに角丸を適用

---

8種類のAndroidアプリテンプレート（Material3 UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Switch/Checkbox](https://zenn.dev/myougatheaxo/articles/android-compose-compose-switch-2026)
- [Chip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-chip-2026)
- [Button](https://zenn.dev/myougatheaxo/articles/android-compose-compose-button-variants-2026)
