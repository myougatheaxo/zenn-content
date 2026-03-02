---
title: "Compose FilledTonalButton完全ガイド — Tonal配色/SegmentedButton/ButtonGroup"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose FilledTonalButton**（FilledTonalButton、SegmentedButton、ButtonGroup、Tonal配色パターン）を解説します。

---

## FilledTonalButton

```kotlin
@Composable
fun TonalButtonDemo() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        // 基本
        FilledTonalButton(onClick = {}) { Text("フィルタ適用") }

        // アイコン付き
        FilledTonalButton(onClick = {}) {
            Icon(Icons.Default.FilterList, null, Modifier.size(ButtonDefaults.IconSize))
            Spacer(Modifier.size(ButtonDefaults.IconSpacing))
            Text("フィルタ")
        }

        // 無効化
        FilledTonalButton(onClick = {}, enabled = false) { Text("処理中...") }
    }
}
```

---

## SegmentedButton

```kotlin
@Composable
fun SegmentedButtonDemo() {
    var selectedIndex by remember { mutableIntStateOf(0) }
    val options = listOf("日", "週", "月")

    SingleChoiceSegmentedButtonRow(Modifier.fillMaxWidth()) {
        options.forEachIndexed { index, label ->
            SegmentedButton(
                selected = index == selectedIndex,
                onClick = { selectedIndex = index },
                shape = SegmentedButtonDefaults.itemShape(index, options.size)
            ) { Text(label) }
        }
    }
}

// MultiChoice
@Composable
fun MultiSegmentedButtonDemo() {
    val selected = remember { mutableStateMapOf("太字" to false, "斜体" to false, "下線" to false) }

    MultiChoiceSegmentedButtonRow(Modifier.fillMaxWidth()) {
        selected.entries.forEachIndexed { index, (label, isSelected) ->
            SegmentedButton(
                checked = isSelected,
                onCheckedChange = { selected[label] = it },
                shape = SegmentedButtonDefaults.itemShape(index, selected.size)
            ) { Text(label) }
        }
    }
}
```

---

## ボタングループパターン

```kotlin
@Composable
fun ButtonGroupPattern() {
    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        // アクションバー
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            OutlinedButton(onClick = {}, Modifier.weight(1f)) { Text("キャンセル") }
            FilledTonalButton(onClick = {}, Modifier.weight(1f)) { Text("下書き") }
            Button(onClick = {}, Modifier.weight(1f)) { Text("公開") }
        }

        // ステッパー
        Row(verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            FilledTonalIconButton(onClick = {}) { Icon(Icons.Default.Remove, "減らす") }
            Text("3", style = MaterialTheme.typography.titleLarge)
            FilledTonalIconButton(onClick = {}) { Icon(Icons.Default.Add, "増やす") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `FilledTonalButton` | 中強調ボタン |
| `SegmentedButton` | 選択肢切替 |
| `SingleChoiceSegmentedButtonRow` | 単一選択 |
| `MultiChoiceSegmentedButtonRow` | 複数選択 |

- `FilledTonalButton`はsecondaryContainer色で中程度の強調
- `SegmentedButton`はタブ的な切替UIに最適
- Single/MultiChoiceで単一・複数選択を切り分け
- `itemShape`でセグメントの角丸を自動計算

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose TextButton](https://zenn.dev/myougatheaxo/articles/android-compose-compose-text-button-2026)
- [Compose IconButton](https://zenn.dev/myougatheaxo/articles/android-compose-compose-icon-button-2026)
- [Compose Chip](https://zenn.dev/myougatheaxo/articles/android-compose-compose-chip-2026)
