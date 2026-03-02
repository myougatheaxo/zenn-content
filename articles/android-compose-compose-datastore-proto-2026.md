---
title: "Compose DataStore Proto完全ガイド — Protocol Buffers/型安全/マイグレーション/Compose連携"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "datastore"]
published: true
---

## この記事で学べること

**Compose DataStore Proto**（Protocol Buffers定義、型安全なデータ保存、マイグレーション、Compose状態連携）を解説します。

---

## Proto定義

```protobuf
// app/src/main/proto/user_preferences.proto
syntax = "proto3";

option java_package = "com.example.app.data";

message UserPreferences {
    bool dark_mode = 1;
    string language = 2;
    int32 font_size = 3;
    SortOrder sort_order = 4;

    enum SortOrder {
        UNSPECIFIED = 0;
        BY_NAME = 1;
        BY_DATE = 2;
        BY_SIZE = 3;
    }
}
```

---

## DataStore実装

```kotlin
object UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()
    override suspend fun readFrom(input: InputStream): UserPreferences =
        UserPreferences.parseFrom(input)
    override suspend fun writeTo(t: UserPreferences, output: OutputStream) =
        t.writeTo(output)
}

// DI
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    @Provides
    @Singleton
    fun provideProtoDataStore(@ApplicationContext context: Context): DataStore<UserPreferences> =
        DataStoreFactory.create(
            serializer = UserPreferencesSerializer,
            produceFile = { context.dataStoreFile("user_preferences.pb") }
        )
}

// Repository
class SettingsRepository @Inject constructor(
    private val dataStore: DataStore<UserPreferences>
) {
    val preferences: Flow<UserPreferences> = dataStore.data

    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.updateData { it.toBuilder().setDarkMode(enabled).build() }
    }

    suspend fun setLanguage(language: String) {
        dataStore.updateData { it.toBuilder().setLanguage(language).build() }
    }

    suspend fun setSortOrder(order: UserPreferences.SortOrder) {
        dataStore.updateData { it.toBuilder().setSortOrder(order).build() }
    }
}
```

---

## Compose連携

```kotlin
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val repository: SettingsRepository
) : ViewModel() {
    val preferences = repository.preferences
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UserPreferences.getDefaultInstance())

    fun toggleDarkMode() = viewModelScope.launch {
        repository.setDarkMode(!preferences.value.darkMode)
    }

    fun setLanguage(lang: String) = viewModelScope.launch {
        repository.setLanguage(lang)
    }
}

@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val prefs by viewModel.preferences.collectAsStateWithLifecycle()

    LazyColumn(Modifier.padding(16.dp)) {
        item {
            ListItem(
                headlineContent = { Text("ダークモード") },
                trailingContent = {
                    Switch(checked = prefs.darkMode, onCheckedChange = { viewModel.toggleDarkMode() })
                }
            )
        }
        item {
            ListItem(
                headlineContent = { Text("言語: ${prefs.language.ifEmpty { "日本語" }}") },
                supportingContent = { Text("タップして変更") }
            )
        }
        item {
            ListItem(
                headlineContent = { Text("フォントサイズ: ${prefs.fontSize}") },
                supportingContent = { Text("ソート: ${prefs.sortOrder.name}") }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| Proto DataStore | 型安全データ保存 |
| `Serializer` | シリアライズ定義 |
| `updateData` | データ更新 |
| Protocol Buffers | スキーマ定義 |

- Protocol Buffersで型安全なスキーマ定義
- `Serializer`でシリアライズ/デシリアライズを実装
- `updateData`でアトミックな更新
- Preferences DataStoreより型安全、複雑なデータに最適

---

8種類のAndroidアプリテンプレート（DataStore対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-compose-datastore-2026)
- [Compose Encryption](https://zenn.dev/myougatheaxo/articles/android-compose-compose-encryption-2026)
- [Compose Room](https://zenn.dev/myougatheaxo/articles/android-compose-compose-room-2026)
