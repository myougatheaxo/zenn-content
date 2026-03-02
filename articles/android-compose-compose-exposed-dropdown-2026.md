---
title: "ExposedDropdownMenu完全ガイド — フィルタ/検索/フォーム選択"
emoji: "🔽"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**ExposedDropdownMenu**（フォーム選択、フィルタ検索、読み取り専用ドロップダウン）を解説します。

---

## 基本ExposedDropdownMenu

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun BasicExposedDropdown() {
    val options = listOf("日本語", "English", "中文", "한국어")
    var expanded by remember { mutableStateOf(false) }
    var selected by remember { mutableStateOf(options[0]) }

    ExposedDropdownMenuBox(expanded = expanded, onExpandedChange = { expanded = it }) {
        OutlinedTextField(
            value = selected,
            onValueChange = {},
            readOnly = true,
            label = { Text("言語") },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
            modifier = Modifier.menuAnchor(MenuAnchorType.PrimaryNotEditable).fillMaxWidth()
        )
        ExposedDropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
            options.forEach { option ->
                DropdownMenuItem(
                    text = { Text(option) },
                    onClick = {
                        selected = option
                        expanded = false
                    },
                    contentPadding = ExposedDropdownMenuDefaults.ItemContentPadding
                )
            }
        }
    }
}
```

---

## フィルタ付き

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FilterableDropdown() {
    val allOptions = listOf("東京", "大阪", "名古屋", "福岡", "札幌", "仙台", "横浜", "神戸")
    var expanded by remember { mutableStateOf(false) }
    var query by remember { mutableStateOf("") }
    val filtered = allOptions.filter { it.contains(query, ignoreCase = true) }

    ExposedDropdownMenuBox(expanded = expanded, onExpandedChange = { expanded = it }) {
        OutlinedTextField(
            value = query,
            onValueChange = {
                query = it
                expanded = true
            },
            label = { Text("都市を検索") },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
            modifier = Modifier.menuAnchor(MenuAnchorType.PrimaryEditable).fillMaxWidth()
        )
        if (filtered.isNotEmpty()) {
            ExposedDropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
                filtered.forEach { option ->
                    DropdownMenuItem(
                        text = { Text(option) },
                        onClick = {
                            query = option
                            expanded = false
                        }
                    )
                }
            }
        }
    }
}
```

---

## フォーム内使用

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RegistrationForm() {
    var name by remember { mutableStateOf("") }
    val prefectures = listOf("北海道", "東京都", "大阪府", "京都府", "福岡県")
    var selectedPrefecture by remember { mutableStateOf("") }
    var prefExpanded by remember { mutableStateOf(false) }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        OutlinedTextField(
            value = name, onValueChange = { name = it },
            label = { Text("名前") }, modifier = Modifier.fillMaxWidth()
        )

        ExposedDropdownMenuBox(expanded = prefExpanded, onExpandedChange = { prefExpanded = it }) {
            OutlinedTextField(
                value = selectedPrefecture,
                onValueChange = {},
                readOnly = true,
                label = { Text("都道府県") },
                trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(prefExpanded) },
                modifier = Modifier.menuAnchor(MenuAnchorType.PrimaryNotEditable).fillMaxWidth()
            )
            ExposedDropdownMenu(expanded = prefExpanded, onDismissRequest = { prefExpanded = false }) {
                prefectures.forEach { pref ->
                    DropdownMenuItem(
                        text = { Text(pref) },
                        onClick = { selectedPrefecture = pref; prefExpanded = false }
                    )
                }
            }
        }

        Button(onClick = {}, Modifier.fillMaxWidth()) { Text("登録") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ExposedDropdownMenuBox` | ドロップダウンコンテナ |
| `menuAnchor()` | TextFieldとメニューの接続 |
| `readOnly = true` | 選択のみ（編集不可） |
| `TrailingIcon` | 展開/折り畳みアイコン |

- `ExposedDropdownMenu`でフォーム向け選択UI
- `readOnly`で選択専用、`editable`で検索可能
- `menuAnchor`でTextField直下にメニュー表示
- バリデーションとの組み合わせでフォーム完成

---

8種類のAndroidアプリテンプレート（フォーム対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DropdownMenu](https://zenn.dev/myougatheaxo/articles/android-compose-compose-dropdown-menu-2026)
- [TextField/バリデーション](https://zenn.dev/myougatheaxo/articles/android-compose-textfield-validation-2026)
- [SearchBar](https://zenn.dev/myougatheaxo/articles/android-compose-compose-search-bar-2026)
