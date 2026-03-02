---
title: "Preferences DataStore完全ガイド — SharedPreferences移行"
emoji: "💾"
type: "tech"
topics: ["android", "kotlin", "datastore", "preferences"]
published: true
---

## この記事で学べること

**Preferences DataStore**（設定保存、Flow連携、SharedPreferences移行、テスト）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.datastore:datastore-preferences:1.1.1")
}

// DataStoreインスタンス（トップレベルで1つだけ）
val Context.dataStore by preferencesDataStore(name = "settings")
```

---

## キー定義と読み書き

```kotlin
object PrefsKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val FONT_SIZE = intPreferencesKey("font_size")
    val USERNAME = stringPreferencesKey("username")
    val LANGUAGE = stringPreferencesKey("language")
    val NOTIFICATION_ENABLED = booleanPreferencesKey("notification_enabled")
    val LAST_SYNC = longPreferencesKey("last_sync")
}

class SettingsRepository(private val dataStore: DataStore<Preferences>) {

    // 読み取り（Flow）
    val darkMode: Flow<Boolean> = dataStore.data.map { prefs ->
        prefs[PrefsKeys.DARK_MODE] ?: false
    }

    val fontSize: Flow<Int> = dataStore.data.map { prefs ->
        prefs[PrefsKeys.FONT_SIZE] ?: 16
    }

    val settings: Flow<AppSettings> = dataStore.data.map { prefs ->
        AppSettings(
            darkMode = prefs[PrefsKeys.DARK_MODE] ?: false,
            fontSize = prefs[PrefsKeys.FONT_SIZE] ?: 16,
            language = prefs[PrefsKeys.LANGUAGE] ?: "ja",
            notificationEnabled = prefs[PrefsKeys.NOTIFICATION_ENABLED] ?: true
        )
    }

    // 書き込み
    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { prefs ->
            prefs[PrefsKeys.DARK_MODE] = enabled
        }
    }

    suspend fun setFontSize(size: Int) {
        dataStore.edit { prefs ->
            prefs[PrefsKeys.FONT_SIZE] = size
        }
    }

    suspend fun updateSettings(settings: AppSettings) {
        dataStore.edit { prefs ->
            prefs[PrefsKeys.DARK_MODE] = settings.darkMode
            prefs[PrefsKeys.FONT_SIZE] = settings.fontSize
            prefs[PrefsKeys.LANGUAGE] = settings.language
            prefs[PrefsKeys.NOTIFICATION_ENABLED] = settings.notificationEnabled
        }
    }

    suspend fun clearAll() {
        dataStore.edit { it.clear() }
    }
}
```

---

## ViewModel連携

```kotlin
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val repository: SettingsRepository
) : ViewModel() {

    val settings = repository.settings
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = AppSettings()
        )

    fun toggleDarkMode() {
        viewModelScope.launch {
            repository.setDarkMode(!settings.value.darkMode)
        }
    }

    fun setFontSize(size: Int) {
        viewModelScope.launch {
            repository.setFontSize(size)
        }
    }
}
```

---

## Compose設定画面

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    LazyColumn(Modifier.padding(16.dp)) {
        item {
            Text("設定", style = MaterialTheme.typography.headlineMedium)
            Spacer(Modifier.height(16.dp))
        }

        // ダークモード
        item {
            Row(
                Modifier.fillMaxWidth(),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Column(Modifier.weight(1f)) {
                    Text("ダークモード")
                    Text("暗い配色に切り替え", style = MaterialTheme.typography.bodySmall)
                }
                Switch(
                    checked = settings.darkMode,
                    onCheckedChange = { viewModel.toggleDarkMode() }
                )
            }
            HorizontalDivider(Modifier.padding(vertical = 8.dp))
        }

        // フォントサイズ
        item {
            Text("フォントサイズ: ${settings.fontSize}sp")
            Slider(
                value = settings.fontSize.toFloat(),
                onValueChange = { viewModel.setFontSize(it.toInt()) },
                valueRange = 12f..24f,
                steps = 5
            )
        }
    }
}
```

---

## SharedPreferences移行

```kotlin
// 既存のSharedPreferencesから移行
val Context.dataStore by preferencesDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(SharedPreferencesMigration(context, "old_prefs"))
    }
)

// 移行完了後、old_prefsは自動削除される
```

---

## Hilt Module

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {

    @Provides
    @Singleton
    fun provideDataStore(@ApplicationContext context: Context): DataStore<Preferences> {
        return context.dataStore
    }

    @Provides
    @Singleton
    fun provideSettingsRepository(dataStore: DataStore<Preferences>): SettingsRepository {
        return SettingsRepository(dataStore)
    }
}
```

---

## まとめ

- `preferencesDataStore`でインスタンス作成（シングルトン）
- `Flow<Preferences>`でリアクティブに値を監視
- `dataStore.edit { }`でトランザクション的に書き込み
- `SharedPreferencesMigration`で既存SPから移行
- `SharingStarted.WhileSubscribed(5000)`で画面回転対応
- 型安全なキーで`Int`/`String`/`Boolean`/`Long`/`Float`/`Double`/`Set<String>`をサポート

---

8種類のAndroidアプリテンプレート（DataStore設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Proto DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-proto-datastore-2026)
- [設定画面](https://zenn.dev/myougatheaxo/articles/android-compose-settings-screen-2026)
- [テーマ切替](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
