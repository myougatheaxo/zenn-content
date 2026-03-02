---
title: "Compose Switch完全ガイド — M3 Switch/アイコン付き/カスタムカラー"
emoji: "🔘"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Switch**（M3 Switch、アイコン付きSwitch、カスタムカラー、設定画面パターン）を解説します。

---

## 基本Switch

```kotlin
@Composable
fun SwitchDemo() {
    var checked by remember { mutableStateOf(false) }

    Row(Modifier.fillMaxWidth().padding(16.dp), horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically) {
        Text("通知")
        Switch(checked = checked, onCheckedChange = { checked = it })
    }
}

// アイコン付き
@Composable
fun IconSwitchDemo() {
    var checked by remember { mutableStateOf(true) }

    Switch(
        checked = checked,
        onCheckedChange = { checked = it },
        thumbContent = {
            Icon(
                imageVector = if (checked) Icons.Default.Check else Icons.Default.Close,
                contentDescription = null,
                modifier = Modifier.size(SwitchDefaults.IconSize)
            )
        }
    )
}
```

---

## 設定画面パターン

```kotlin
@Composable
fun SettingsToggle(
    title: String,
    description: String? = null,
    icon: ImageVector? = null,
    checked: Boolean,
    onCheckedChange: (Boolean) -> Unit
) {
    ListItem(
        headlineContent = { Text(title) },
        supportingContent = description?.let { { Text(it) } },
        leadingContent = icon?.let { { Icon(it, null) } },
        trailingContent = { Switch(checked = checked, onCheckedChange = onCheckedChange) },
        modifier = Modifier.clickable { onCheckedChange(!checked) }
    )
}

@Composable
fun SettingsScreen() {
    var darkMode by remember { mutableStateOf(false) }
    var notifications by remember { mutableStateOf(true) }
    var autoSync by remember { mutableStateOf(true) }

    Column {
        SettingsToggle("ダークモード", "外観を切替", Icons.Default.DarkMode, darkMode) { darkMode = it }
        HorizontalDivider()
        SettingsToggle("プッシュ通知", "新着通知を受信", Icons.Default.Notifications, notifications) { notifications = it }
        HorizontalDivider()
        SettingsToggle("自動同期", "Wi-Fi接続時に同期", Icons.Default.Sync, autoSync) { autoSync = it }
    }
}
```

---

## カスタムカラー

```kotlin
@Composable
fun CustomColorSwitch() {
    var checked by remember { mutableStateOf(false) }

    Switch(
        checked = checked,
        onCheckedChange = { checked = it },
        colors = SwitchDefaults.colors(
            checkedThumbColor = Color.White,
            checkedTrackColor = Color(0xFF4CAF50),
            uncheckedThumbColor = Color.Gray,
            uncheckedTrackColor = Color.LightGray
        )
    )
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `Switch` | ON/OFFトグル |
| `thumbContent` | アイコン付きサム |
| `SwitchDefaults.colors` | カラーカスタマイズ |
| `ListItem` + `Switch` | 設定画面パターン |

- M3 Switchはアイコン付きサムに対応
- `ListItem`と組み合わせて設定画面に最適
- `clickable`でリスト全体をタップ可能に
- `SwitchDefaults.colors`でブランドカラーに統一

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose CheckboxGroup](https://zenn.dev/myougatheaxo/articles/android-compose-compose-checkbox-group-2026)
- [Compose RadioGroup](https://zenn.dev/myougatheaxo/articles/android-compose-compose-radio-group-2026)
- [Compose SettingsScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-settings-screen-2026)
