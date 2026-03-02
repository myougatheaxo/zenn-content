---
title: "ポップアップメニュー完全ガイド — DropdownMenu/ExposedDropdown/Popup"
emoji: "📋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**ポップアップメニュー**（DropdownMenu、ExposedDropdownMenuBox、Popup、カスタムメニュー）を解説します。

---

## DropdownMenu

```kotlin
@Composable
fun OverflowMenu(onAction: (String) -> Unit) {
    var expanded by remember { mutableStateOf(false) }

    Box {
        IconButton(onClick = { expanded = true }) {
            Icon(Icons.Default.MoreVert, "メニュー")
        }

        DropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            DropdownMenuItem(
                text = { Text("編集") },
                leadingIcon = { Icon(Icons.Default.Edit, null) },
                onClick = { onAction("edit"); expanded = false }
            )
            DropdownMenuItem(
                text = { Text("共有") },
                leadingIcon = { Icon(Icons.Default.Share, null) },
                onClick = { onAction("share"); expanded = false }
            )
            HorizontalDivider()
            DropdownMenuItem(
                text = { Text("削除", color = MaterialTheme.colorScheme.error) },
                leadingIcon = { Icon(Icons.Default.Delete, null, tint = MaterialTheme.colorScheme.error) },
                onClick = { onAction("delete"); expanded = false }
            )
        }
    }
}
```

---

## ExposedDropdownMenuBox

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CategorySelector(
    categories: List<String>,
    selected: String,
    onSelect: (String) -> Unit
) {
    var expanded by remember { mutableStateOf(false) }

    ExposedDropdownMenuBox(
        expanded = expanded,
        onExpandedChange = { expanded = it }
    ) {
        OutlinedTextField(
            value = selected,
            onValueChange = {},
            readOnly = true,
            label = { Text("カテゴリ") },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
            colors = ExposedDropdownMenuDefaults.outlinedTextFieldColors(),
            modifier = Modifier.menuAnchor().fillMaxWidth()
        )

        ExposedDropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            categories.forEach { category ->
                DropdownMenuItem(
                    text = { Text(category) },
                    onClick = {
                        onSelect(category)
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

## Popup

```kotlin
@Composable
fun CustomPopup(show: Boolean, onDismiss: () -> Unit) {
    if (show) {
        Popup(
            alignment = Alignment.Center,
            onDismissRequest = onDismiss
        ) {
            Surface(
                shape = RoundedCornerShape(16.dp),
                tonalElevation = 8.dp,
                shadowElevation = 8.dp,
                modifier = Modifier.padding(16.dp)
            ) {
                Column(Modifier.padding(24.dp)) {
                    Text("カスタムポップアップ", style = MaterialTheme.typography.titleMedium)
                    Spacer(Modifier.height(8.dp))
                    Text("自由なレイアウトで表示できます")
                    Spacer(Modifier.height(16.dp))
                    Button(onClick = onDismiss, Modifier.align(Alignment.End)) {
                        Text("閉じる")
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|---------------|------|
| DropdownMenu | オーバーフローメニュー |
| ExposedDropdownMenuBox | フォーム選択 |
| Popup | カスタムオーバーレイ |

- `DropdownMenu`でコンテキストメニュー
- `ExposedDropdownMenuBox`でフォーム選択UI
- `Popup`で自由配置のオーバーレイ
- `HorizontalDivider`でメニュー項目をグループ化

---

8種類のAndroidアプリテンプレート（メニューUI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [コンテキストメニュー](https://zenn.dev/myougatheaxo/articles/android-compose-context-menu-2026)
- [BottomSheet](https://zenn.dev/myougatheaxo/articles/android-compose-bottomsheet-dialog-2026)
- [Tooltip](https://zenn.dev/myougatheaxo/articles/android-compose-tooltip-2026)
