---
title: "Settings Screen完全ガイド — 設定画面UI/PreferenceGroup/DataStore連携"
emoji: "⚙️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "settings"]
published: true
---

## この記事で学べること

**Settings Screen**（設定画面UI、グループ分け、Switch/Slider設定項目、DataStore連携）を解説します。

---

## 設定画面UI

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(vertical = 8.dp)
    ) {
        item { SettingsGroupTitle("一般") }
        item {
            SwitchSetting(
                title = "ダークモード",
                description = "アプリの外観をダークテーマにします",
                icon = Icons.Default.DarkMode,
                checked = settings.isDarkMode,
                onCheckedChange = { viewModel.setDarkMode(it) }
            )
        }
        item {
            SwitchSetting(
                title = "通知",
                description = "プッシュ通知を受け取る",
                icon = Icons.Default.Notifications,
                checked = settings.notificationsEnabled,
                onCheckedChange = { viewModel.setNotifications(it) }
            )
        }

        item { SettingsGroupTitle("表示") }
        item {
            SliderSetting(
                title = "文字サイズ",
                icon = Icons.Default.TextFields,
                value = settings.fontSize,
                valueRange = 12f..24f,
                onValueChange = { viewModel.setFontSize(it) }
            )
        }

        item { SettingsGroupTitle("アカウント") }
        item {
            ClickableSetting(
                title = "プロフィール編集",
                icon = Icons.Default.Person,
                onClick = { /* navigate */ }
            )
        }
        item {
            ClickableSetting(
                title = "ログアウト",
                icon = Icons.Default.Logout,
                onClick = { /* logout */ },
                textColor = Color.Red
            )
        }
    }
}
```

---

## 設定コンポーネント

```kotlin
@Composable
fun SettingsGroupTitle(title: String) {
    Text(
        title,
        style = MaterialTheme.typography.labelLarge,
        color = MaterialTheme.colorScheme.primary,
        modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp)
    )
}

@Composable
fun SwitchSetting(
    title: String, description: String? = null,
    icon: ImageVector, checked: Boolean, onCheckedChange: (Boolean) -> Unit
) {
    ListItem(
        headlineContent = { Text(title) },
        supportingContent = description?.let { { Text(it) } },
        leadingContent = { Icon(icon, null) },
        trailingContent = { Switch(checked = checked, onCheckedChange = onCheckedChange) },
        modifier = Modifier.clickable { onCheckedChange(!checked) }
    )
}

@Composable
fun SliderSetting(
    title: String, icon: ImageVector,
    value: Float, valueRange: ClosedFloatingPointRange<Float>,
    onValueChange: (Float) -> Unit
) {
    Column(Modifier.padding(horizontal = 16.dp, vertical = 8.dp)) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(icon, null)
            Spacer(Modifier.width(16.dp))
            Text(title, Modifier.weight(1f))
            Text("${value.toInt()}sp", style = MaterialTheme.typography.bodySmall)
        }
        Slider(value = value, onValueChange = onValueChange, valueRange = valueRange)
    }
}

@Composable
fun ClickableSetting(
    title: String, icon: ImageVector,
    onClick: () -> Unit, textColor: Color = Color.Unspecified
) {
    ListItem(
        headlineContent = { Text(title, color = textColor) },
        leadingContent = { Icon(icon, null, tint = if (textColor != Color.Unspecified) textColor else LocalContentColor.current) },
        trailingContent = { Icon(Icons.AutoMirrored.Filled.ArrowForward, null, Modifier.size(16.dp)) },
        modifier = Modifier.clickable(onClick = onClick)
    )
}
```

---

## DataStore連携

```kotlin
data class AppSettings(
    val isDarkMode: Boolean = false,
    val notificationsEnabled: Boolean = true,
    val fontSize: Float = 16f
)

@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val dataStore: DataStore<Preferences>
) : ViewModel() {
    val settings = dataStore.data.map { prefs ->
        AppSettings(
            isDarkMode = prefs[booleanPreferencesKey("dark_mode")] ?: false,
            notificationsEnabled = prefs[booleanPreferencesKey("notifications")] ?: true,
            fontSize = prefs[floatPreferencesKey("font_size")] ?: 16f
        )
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), AppSettings())

    fun setDarkMode(enabled: Boolean) = update { it[booleanPreferencesKey("dark_mode")] = enabled }
    fun setNotifications(enabled: Boolean) = update { it[booleanPreferencesKey("notifications")] = enabled }
    fun setFontSize(size: Float) = update { it[floatPreferencesKey("font_size")] = size }

    private fun update(block: (MutablePreferences) -> Unit) {
        viewModelScope.launch { dataStore.edit(block) }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `SwitchSetting` | ON/OFF設定 |
| `SliderSetting` | 数値調整 |
| `ClickableSetting` | 画面遷移設定 |
| `DataStore` | 設定値永続化 |

- 再利用可能な設定コンポーネントで統一UI
- `DataStore`で設定値を永続化
- `ListItem`ベースでMaterial3準拠
- グループ分けで見やすい設定画面

---

8種類のAndroidアプリテンプレート（設定画面対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
- [Switch](https://zenn.dev/myougatheaxo/articles/android-compose-compose-switch-2026)
- [Slider](https://zenn.dev/myougatheaxo/articles/android-compose-compose-slider-2026)
