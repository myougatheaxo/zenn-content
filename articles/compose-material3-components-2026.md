---
title: "Material3コンポーネントカタログ — Button・Card・Chip・FABの全パターン"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3で使える主要UIコンポーネントの**全バリエーション**を一覧で紹介します。コピペですぐ使えるコード付き。

---

## Button（5種類）

```kotlin
// Filled（メイン操作）
Button(onClick = { }) { Text("保存") }

// Outlined（セカンダリ操作）
OutlinedButton(onClick = { }) { Text("キャンセル") }

// Text（低優先度操作）
TextButton(onClick = { }) { Text("詳細を見る") }

// Elevated（浮き上がり効果）
ElevatedButton(onClick = { }) { Text("共有") }

// FilledTonal（中間優先度）
FilledTonalButton(onClick = { }) { Text("フィルター") }
```

| ボタン | 用途 |
|--------|------|
| `Button` | 最も重要な操作（保存、送信） |
| `OutlinedButton` | セカンダリ操作（キャンセル） |
| `TextButton` | 最も低い優先度 |
| `ElevatedButton` | 背景から浮かせたい場合 |
| `FilledTonalButton` | 中間の優先度 |

---

## IconButton

```kotlin
IconButton(onClick = { }) {
    Icon(Icons.Default.Favorite, "お気に入り")
}

FilledIconButton(onClick = { }) {
    Icon(Icons.Default.Add, "追加")
}

OutlinedIconButton(onClick = { }) {
    Icon(Icons.Default.Share, "共有")
}
```

---

## Card（3種類）

```kotlin
// 標準
Card(Modifier.fillMaxWidth().padding(8.dp)) {
    Text("Card", Modifier.padding(16.dp))
}

// Elevated（影あり）
ElevatedCard(Modifier.fillMaxWidth().padding(8.dp)) {
    Text("ElevatedCard", Modifier.padding(16.dp))
}

// Outlined（枠線）
OutlinedCard(Modifier.fillMaxWidth().padding(8.dp)) {
    Text("OutlinedCard", Modifier.padding(16.dp))
}
```

---

## Chip（4種類）

```kotlin
// Assist（提案）
AssistChip(
    onClick = { },
    label = { Text("道順を表示") },
    leadingIcon = { Icon(Icons.Default.LocationOn, null) }
)

// Filter（フィルタリング）
var selected by remember { mutableStateOf(false) }
FilterChip(
    selected = selected,
    onClick = { selected = !selected },
    label = { Text("カテゴリA") },
    leadingIcon = if (selected) {
        { Icon(Icons.Default.Check, null) }
    } else null
)

// Input（入力タグ）
InputChip(
    selected = false,
    onClick = { },
    label = { Text("Kotlin") },
    trailingIcon = { Icon(Icons.Default.Close, "削除") }
)

// Suggestion（サジェスト）
SuggestionChip(
    onClick = { },
    label = { Text("おすすめ") }
)
```

---

## FAB（Floating Action Button）

```kotlin
// 通常
FloatingActionButton(onClick = { }) {
    Icon(Icons.Default.Add, "追加")
}

// Small
SmallFloatingActionButton(onClick = { }) {
    Icon(Icons.Default.Add, "追加")
}

// Large
LargeFloatingActionButton(onClick = { }) {
    Icon(Icons.Default.Add, "追加")
}

// Extended（テキスト付き）
ExtendedFloatingActionButton(
    onClick = { },
    icon = { Icon(Icons.Default.Edit, null) },
    text = { Text("新規作成") }
)
```

---

## Switch & Checkbox

```kotlin
// Switch
var checked by remember { mutableStateOf(false) }
Switch(
    checked = checked,
    onCheckedChange = { checked = it }
)

// Checkbox
var isChecked by remember { mutableStateOf(false) }
Row(verticalAlignment = Alignment.CenterVertically) {
    Checkbox(
        checked = isChecked,
        onCheckedChange = { isChecked = it }
    )
    Text("利用規約に同意する")
}
```

---

## LinearProgressIndicator

```kotlin
// 確定的（進捗あり）
LinearProgressIndicator(
    progress = { 0.7f },
    modifier = Modifier.fillMaxWidth()
)

// 不確定的（ローディング）
LinearProgressIndicator(Modifier.fillMaxWidth())

// 円形
CircularProgressIndicator()
```

---

## まとめ

| カテゴリ | コンポーネント |
|---------|-------------|
| ボタン | Button, OutlinedButton, TextButton, ElevatedButton, FilledTonalButton |
| アイコンボタン | IconButton, FilledIconButton, OutlinedIconButton |
| カード | Card, ElevatedCard, OutlinedCard |
| チップ | AssistChip, FilterChip, InputChip, SuggestionChip |
| FAB | FloatingActionButton, SmallFAB, LargeFAB, ExtendedFAB |
| 入力 | Switch, Checkbox, RadioButton |
| 進捗 | LinearProgressIndicator, CircularProgressIndicator |

---

8種類のAndroidアプリテンプレート（Material3コンポーネント活用済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [ダイアログ完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-dialog-bottomsheet-2026)
