---
title: "DataStore完全ガイド — SharedPreferencesからの移行"
emoji: "💾"
type: "tech"
topics: ["android", "kotlin", "datastore", "storage"]
published: true
---

## この記事で学べること

Android**DataStore**（Preferences / Proto）の実装方法とSharedPreferencesからの移行を解説します。

---

## 依存関係

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.datastore:datastore-preferences:1.1.2")
}
```

---

## Preferences DataStore

```kotlin
// DataStoreの作成
val Context.dataStore by preferencesDataStore(name = "settings")

// キー定義
object PreferenceKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val FONT_SIZE = intPreferencesKey("font_size")
    val USERNAME = stringPreferencesKey("username")
    val LANGUAGE = stringPreferencesKey("language")
}
```

---

## 読み書き

```kotlin
class SettingsRepository(private val dataStore: DataStore<Preferences>) {

    // 読み取り（Flow）
    val darkMode: Flow<Boolean> = dataStore.data.map { prefs ->
        prefs[PreferenceKeys.DARK_MODE] ?: false
    }

    val fontSize: Flow<Int> = dataStore.data.map { prefs ->
        prefs[PreferenceKeys.FONT_SIZE] ?: 14
    }

    val settings: Flow<UserSettings> = dataStore.data.map { prefs ->
        UserSettings(
            darkMode = prefs[PreferenceKeys.DARK_MODE] ?: false,
            fontSize = prefs[PreferenceKeys.FONT_SIZE] ?: 14,
            username = prefs[PreferenceKeys.USERNAME] ?: "",
            language = prefs[PreferenceKeys.LANGUAGE] ?: "ja"
        )
    }

    // 書き込み
    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { prefs ->
            prefs[PreferenceKeys.DARK_MODE] = enabled
        }
    }

    suspend fun updateSettings(
        darkMode: Boolean? = null,
        fontSize: Int? = null,
        username: String? = null
    ) {
        dataStore.edit { prefs ->
            darkMode?.let { prefs[PreferenceKeys.DARK_MODE] = it }
            fontSize?.let { prefs[PreferenceKeys.FONT_SIZE] = it }
            username?.let { prefs[PreferenceKeys.USERNAME] = it }
        }
    }

    suspend fun clearAll() {
        dataStore.edit { it.clear() }
    }
}
```

---

## ViewModel

```kotlin
class SettingsViewModel(
    private val repository: SettingsRepository
) : ViewModel() {

    val settings = repository.settings.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = UserSettings()
    )

    fun toggleDarkMode() {
        viewModelScope.launch {
            val current = settings.value.darkMode
            repository.setDarkMode(!current)
        }
    }

    fun updateFontSize(size: Int) {
        viewModelScope.launch {
            repository.updateSettings(fontSize = size)
        }
    }
}

data class UserSettings(
    val darkMode: Boolean = false,
    val fontSize: Int = 14,
    val username: String = "",
    val language: String = "ja"
)
```

---

## Compose UI

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    LazyColumn(Modifier.fillMaxSize()) {
        item {
            ListItem(
                headlineContent = { Text("ダークモード") },
                trailingContent = {
                    Switch(
                        checked = settings.darkMode,
                        onCheckedChange = { viewModel.toggleDarkMode() }
                    )
                }
            )
        }

        item {
            ListItem(
                headlineContent = { Text("フォントサイズ") },
                supportingContent = { Text("${settings.fontSize}pt") },
                trailingContent = {
                    Row {
                        IconButton(onClick = {
                            viewModel.updateFontSize(settings.fontSize - 1)
                        }) {
                            Icon(Icons.Default.Remove, "小さく")
                        }
                        IconButton(onClick = {
                            viewModel.updateFontSize(settings.fontSize + 1)
                        }) {
                            Icon(Icons.Default.Add, "大きく")
                        }
                    }
                }
            )
        }
    }
}
```

---

## SharedPreferencesからの移行

```kotlin
val Context.dataStore by preferencesDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(
            SharedPreferencesMigration(
                context,
                "old_shared_prefs" // 旧SharedPreferencesのファイル名
            )
        )
    }
)
```

---

## まとめ

- `preferencesDataStore`でDataStore作成
- `xxxPreferencesKey()`で型安全なキー定義
- `dataStore.data.map{}`でFlow読み取り
- `dataStore.edit{}`で書き込み（suspend関数）
- ViewModelで`stateIn()`してComposeから収集
- `SharedPreferencesMigration`で自動移行

---

8種類のAndroidアプリテンプレート（DataStore設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Room Database完全ガイド](https://zenn.dev/myougatheaxo/articles/android-room-database-2026)
- [State管理完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-2026)
- [Coroutines & Flowガイド](https://zenn.dev/myougatheaxo/articles/kotlin-coroutines-flow-2026)
