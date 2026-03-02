---
title: "Material3コンポーネント一覧 — Compose対応チートシート"
emoji: "🎨"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Compose **Material3**の主要コンポーネントをカテゴリ別に一覧紹介します。

---

## ボタン系

```kotlin
// Filled Button（主要アクション）
Button(onClick = {}) { Text("ボタン") }

// Outlined Button
OutlinedButton(onClick = {}) { Text("枠線ボタン") }

// Text Button
TextButton(onClick = {}) { Text("テキストボタン") }

// Filled Tonal Button
FilledTonalButton(onClick = {}) { Text("トーナルボタン") }

// Elevated Button
ElevatedButton(onClick = {}) { Text("影付きボタン") }

// Icon Button
IconButton(onClick = {}) { Icon(Icons.Default.Settings, "設定") }

// FAB
FloatingActionButton(onClick = {}) { Icon(Icons.Default.Add, "追加") }
ExtendedFloatingActionButton(onClick = {}, icon = { Icon(Icons.Default.Add, null) }, text = { Text("新規作成") })
SmallFloatingActionButton(onClick = {}) { Icon(Icons.Default.Edit, "編集") }
LargeFloatingActionButton(onClick = {}) { Icon(Icons.Default.Add, "追加") }
```

---

## 入力系

```kotlin
// TextField
TextField(value = text, onValueChange = { text = it }, label = { Text("ラベル") })
OutlinedTextField(value = text, onValueChange = { text = it }, label = { Text("ラベル") })

// Switch
Switch(checked = checked, onCheckedChange = { checked = it })

// Checkbox
Checkbox(checked = checked, onCheckedChange = { checked = it })

// RadioButton
RadioButton(selected = selected, onClick = { selected = true })

// Slider
Slider(value = sliderValue, onValueChange = { sliderValue = it })
RangeSlider(value = range, onValueChange = { range = it })
```

---

## 表示系

```kotlin
// Card
Card(Modifier.fillMaxWidth()) { Column(Modifier.padding(16.dp)) { Text("カード") } }
ElevatedCard(Modifier.fillMaxWidth()) { /* ... */ }
OutlinedCard(Modifier.fillMaxWidth()) { /* ... */ }

// ListItem
ListItem(
    headlineContent = { Text("タイトル") },
    supportingContent = { Text("サブタイトル") },
    leadingContent = { Icon(Icons.Default.Person, null) },
    trailingContent = { Text("詳細") }
)

// Badge
BadgedBox(badge = { Badge { Text("5") } }) {
    Icon(Icons.Default.Notifications, null)
}

// Divider
HorizontalDivider()
VerticalDivider()

// ProgressIndicator
CircularProgressIndicator()
LinearProgressIndicator()
LinearProgressIndicator(progress = { 0.7f })
```

---

## ナビゲーション系

```kotlin
// Top App Bar
TopAppBar(title = { Text("タイトル") })
CenterAlignedTopAppBar(title = { Text("中央タイトル") })
MediumTopAppBar(title = { Text("中サイズ") })
LargeTopAppBar(title = { Text("大サイズ") })

// Bottom Navigation
NavigationBar { NavigationBarItem(selected = true, onClick = {}, icon = { Icon(Icons.Default.Home, null) }, label = { Text("ホーム") }) }

// Navigation Rail
NavigationRail { NavigationRailItem(selected = true, onClick = {}, icon = { Icon(Icons.Default.Home, null) }, label = { Text("ホーム") }) }

// Navigation Drawer
ModalNavigationDrawer(drawerContent = { ModalDrawerSheet { /* items */ } }) { /* content */ }
```

---

## ダイアログ/シート

```kotlin
// AlertDialog
AlertDialog(onDismissRequest = {}, title = { Text("タイトル") }, text = { Text("内容") }, confirmButton = { TextButton(onClick = {}) { Text("OK") } })

// BottomSheet
ModalBottomSheet(onDismissRequest = {}) { /* content */ }

// DatePicker
DatePickerDialog(onDismissRequest = {}, confirmButton = { TextButton(onClick = {}) { Text("OK") } }) { DatePicker(state = rememberDatePickerState()) }
```

---

## まとめ

- Material3は**色/形状/タイポグラフィ**が一貫したデザインシステム
- ボタンは5種類（重要度で使い分け）
- Card/ListItemで情報表示の統一感
- TopAppBarは4種類（画面の重要度で選択）
- `MaterialTheme.colorScheme`でテーマカラー参照
- 全コンポーネントがダークモード/Dynamic Colorに対応

---

8種類のAndroidアプリテンプレート（Material3完全対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [テーマ切り替え](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
- [Material3アイコン](https://zenn.dev/myougatheaxo/articles/android-compose-material3-icons-2026)
- [Chip/FilterChip](https://zenn.dev/myougatheaxo/articles/android-compose-chip-selection-2026)
