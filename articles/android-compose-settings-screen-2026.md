---
title: "設定画面実装ガイド — Switch/Slider/選択ダイアログの設定UI"
emoji: "⚙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "settings"]
published: true
---

## この記事で学べること

Composeでの**設定画面**（Switch、Slider、選択ダイアログ）の実装パターンを解説します。

---

## 基本の設定画面

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    LazyColumn(Modifier.fillMaxSize()) {
        // セクション: 表示
        item {
            Text(
                "表示",
                Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                style = MaterialTheme.typography.titleSmall,
                color = MaterialTheme.colorScheme.primary
            )
        }

        item {
            SwitchSetting(
                title = "ダークモード",
                description = "暗いテーマに切り替え",
                checked = settings.darkMode,
                onCheckedChange = { viewModel.setDarkMode(it) }
            )
        }

        item {
            SliderSetting(
                title = "フォントサイズ",
                value = settings.fontSize.toFloat(),
                valueRange = 12f..24f,
                steps = 5,
                valueLabel = "${settings.fontSize}pt",
                onValueChange = { viewModel.setFontSize(it.toInt()) }
            )
        }

        item { HorizontalDivider() }

        // セクション: 通知
        item {
            Text(
                "通知",
                Modifier.padding(horizontal = 16.dp, vertical = 8.dp),
                style = MaterialTheme.typography.titleSmall,
                color = MaterialTheme.colorScheme.primary
            )
        }

        item {
            SwitchSetting(
                title = "プッシュ通知",
                description = "新しいメッセージの通知",
                checked = settings.pushNotifications,
                onCheckedChange = { viewModel.setPushNotifications(it) }
            )
        }

        item {
            SwitchSetting(
                title = "サウンド",
                description = "通知音を鳴らす",
                checked = settings.notificationSound,
                onCheckedChange = { viewModel.setNotificationSound(it) }
            )
        }

        item { HorizontalDivider() }

        // セクション: その他
        item {
            ClickableSetting(
                title = "言語",
                value = settings.language,
                onClick = { /* 選択ダイアログ */ }
            )
        }

        item {
            ClickableSetting(
                title = "バージョン",
                value = "1.0.0",
                onClick = null
            )
        }
    }
}
```

---

## 設定コンポーネント

```kotlin
@Composable
fun SwitchSetting(
    title: String,
    description: String? = null,
    checked: Boolean,
    onCheckedChange: (Boolean) -> Unit
) {
    ListItem(
        headlineContent = { Text(title) },
        supportingContent = description?.let { { Text(it) } },
        trailingContent = {
            Switch(checked = checked, onCheckedChange = onCheckedChange)
        },
        modifier = Modifier.clickable { onCheckedChange(!checked) }
    )
}

@Composable
fun SliderSetting(
    title: String,
    value: Float,
    valueRange: ClosedFloatingPointRange<Float>,
    steps: Int = 0,
    valueLabel: String,
    onValueChange: (Float) -> Unit
) {
    Column(Modifier.padding(horizontal = 16.dp, vertical = 8.dp)) {
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Text(title, style = MaterialTheme.typography.bodyLarge)
            Text(valueLabel, style = MaterialTheme.typography.bodyMedium)
        }
        Slider(
            value = value,
            onValueChange = onValueChange,
            valueRange = valueRange,
            steps = steps
        )
    }
}

@Composable
fun ClickableSetting(
    title: String,
    value: String,
    onClick: (() -> Unit)?
) {
    ListItem(
        headlineContent = { Text(title) },
        trailingContent = {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(
                    value,
                    color = MaterialTheme.colorScheme.outline,
                    style = MaterialTheme.typography.bodyMedium
                )
                if (onClick != null) {
                    Icon(Icons.Default.ChevronRight, null)
                }
            }
        },
        modifier = if (onClick != null) Modifier.clickable(onClick = onClick) else Modifier
    )
}
```

---

## 選択ダイアログ

```kotlin
@Composable
fun LanguageSelector(
    currentLanguage: String,
    onLanguageSelected: (String) -> Unit
) {
    var showDialog by remember { mutableStateOf(false) }
    val languages = listOf("日本語", "English", "中文", "한국어")

    ClickableSetting(
        title = "言語",
        value = currentLanguage,
        onClick = { showDialog = true }
    )

    if (showDialog) {
        AlertDialog(
            onDismissRequest = { showDialog = false },
            title = { Text("言語を選択") },
            text = {
                Column {
                    languages.forEach { lang ->
                        Row(
                            Modifier
                                .fillMaxWidth()
                                .clickable {
                                    onLanguageSelected(lang)
                                    showDialog = false
                                }
                                .padding(vertical = 12.dp),
                            verticalAlignment = Alignment.CenterVertically
                        ) {
                            RadioButton(
                                selected = lang == currentLanguage,
                                onClick = {
                                    onLanguageSelected(lang)
                                    showDialog = false
                                }
                            )
                            Spacer(Modifier.width(8.dp))
                            Text(lang)
                        }
                    }
                }
            },
            confirmButton = {
                TextButton(onClick = { showDialog = false }) { Text("閉じる") }
            }
        )
    }
}
```

---

## まとめ

- `ListItem` + `Switch`でトグル設定
- `Slider`で範囲値の設定
- `ClickableSetting`で選択ダイアログへの導線
- セクション分けで`Text`ヘッダー + `HorizontalDivider`
- DataStoreと連携して設定値を永続化
- コンポーネントを共通化して再利用

---

8種類のAndroidアプリテンプレート（設定画面設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore完全ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-2026)
- [ダイアログ実装ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-dialog-2026)
- [ダークモード対応ガイド](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
