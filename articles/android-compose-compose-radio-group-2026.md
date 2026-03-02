---
title: "Compose RadioGroup完全ガイド — RadioButton/グループ選択/ListItem連携"
emoji: "🔘"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose RadioGroup**（RadioButton、ラジオグループ、ListItem連携、設定画面パターン）を解説します。

---

## 基本RadioGroup

```kotlin
@Composable
fun RadioGroupDemo() {
    val options = listOf("ライト", "ダーク", "システム設定に従う")
    var selected by remember { mutableStateOf(options[2]) }

    Column(Modifier.padding(16.dp)) {
        Text("テーマ設定", style = MaterialTheme.typography.titleMedium)
        Spacer(Modifier.height(8.dp))

        options.forEach { option ->
            Row(
                Modifier.fillMaxWidth()
                    .selectable(selected = (option == selected),
                        onClick = { selected = option }, role = Role.RadioButton)
                    .padding(vertical = 8.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                RadioButton(selected = (option == selected), onClick = null)
                Spacer(Modifier.width(12.dp))
                Text(option, style = MaterialTheme.typography.bodyLarge)
            }
        }
    }
}
```

---

## ListItem + RadioButton

```kotlin
@Composable
fun RadioListItemDemo() {
    data class SortOption(val label: String, val description: String)
    val options = listOf(
        SortOption("新着順", "最新のアイテムを先に表示"),
        SortOption("人気順", "評価の高いアイテムを先に表示"),
        SortOption("価格順", "安い順に表示"),
        SortOption("名前順", "アルファベット順に表示")
    )
    var selected by remember { mutableStateOf(options[0].label) }

    Column {
        options.forEach { option ->
            ListItem(
                headlineContent = { Text(option.label) },
                supportingContent = { Text(option.description) },
                leadingContent = {
                    RadioButton(selected = (option.label == selected), onClick = null)
                },
                modifier = Modifier.clickable { selected = option.label }
            )
            HorizontalDivider()
        }
    }
}
```

---

## ダイアログ内ラジオグループ

```kotlin
@Composable
fun RadioDialogDemo() {
    var showDialog by remember { mutableStateOf(false) }
    var selectedLanguage by remember { mutableStateOf("日本語") }
    val languages = listOf("日本語", "English", "中文", "한국어")

    ListItem(
        headlineContent = { Text("言語") },
        supportingContent = { Text(selectedLanguage) },
        leadingContent = { Icon(Icons.Default.Language, null) },
        modifier = Modifier.clickable { showDialog = true }
    )

    if (showDialog) {
        var tempSelection by remember { mutableStateOf(selectedLanguage) }
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("言語を選択") },
            text = {
                Column {
                    languages.forEach { lang ->
                        Row(Modifier.fillMaxWidth()
                            .selectable(selected = (lang == tempSelection),
                                onClick = { tempSelection = lang }, role = Role.RadioButton)
                            .padding(vertical = 8.dp),
                            verticalAlignment = Alignment.CenterVertically) {
                            RadioButton(selected = (lang == tempSelection), onClick = null)
                            Spacer(Modifier.width(12.dp))
                            Text(lang)
                        }
                    }
                }
            },
            confirmButton = {
                TextButton(onClick = { selectedLanguage = tempSelection; showDialog = false }) {
                    Text("OK")
                }
            },
            dismissButton = {
                TextButton(onClick = { showDialog = false }) { Text("キャンセル") }
            }
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `RadioButton` | ラジオボタン |
| `selectable` | 行全体を選択可能に |
| `Role.RadioButton` | アクセシビリティ |
| `ListItem` + `RadioButton` | 設定画面パターン |

- `RadioButton`の`onClick = null`で行全体クリックに委譲
- `selectable`修飾子で行全体を選択可能に（アクセシビリティ対応）
- `ListItem`と組み合わせて設定画面の選択UIに
- ダイアログ内でソート順・言語選択等に活用

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose CheckboxGroup](https://zenn.dev/myougatheaxo/articles/android-compose-compose-checkbox-group-2026)
- [Compose Switch](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-switch-2026)
- [Compose Dialog](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dialog-2026)
