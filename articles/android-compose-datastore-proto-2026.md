---
title: "Proto DataStore完全ガイド — 型安全設定/スキーマ定義/Migration"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "datastore"]
published: true
---

## この記事で学べること

**Proto DataStore**（Protocol Buffers定義、型安全な設定管理、Serializer、Migration）を解説します。

---

## Proto定義

```protobuf
// app/src/main/proto/settings.proto
syntax = "proto3";
option java_package = "com.example.app";

message AppSettings {
  string theme = 1;
  bool notifications_enabled = 2;
  int32 font_size = 3;
  string language = 4;
  UserProfile profile = 5;
}

message UserProfile {
  string name = 1;
  string email = 2;
  string avatar_url = 3;
}
```

---

## Serializer

```kotlin
object AppSettingsSerializer : Serializer<AppSettings> {
    override val defaultValue: AppSettings = AppSettings.getDefaultInstance()
    override suspend fun readFrom(input: InputStream): AppSettings =
        try { AppSettings.parseFrom(input) }
        catch (e: InvalidProtocolBufferException) { defaultValue }
    override suspend fun writeTo(t: AppSettings, output: OutputStream) = t.writeTo(output)
}

// DataStore生成
val Context.settingsDataStore by dataStore(
    fileName = "settings.pb",
    serializer = AppSettingsSerializer
)

// Repository
class SettingsRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    val settings: Flow<AppSettings> = context.settingsDataStore.data

    suspend fun updateTheme(theme: String) {
        context.settingsDataStore.updateData { it.toBuilder().setTheme(theme).build() }
    }

    suspend fun setNotifications(enabled: Boolean) {
        context.settingsDataStore.updateData { it.toBuilder().setNotificationsEnabled(enabled).build() }
    }

    suspend fun updateProfile(name: String, email: String) {
        context.settingsDataStore.updateData {
            it.toBuilder().setProfile(
                UserProfile.newBuilder().setName(name).setEmail(email).build()
            ).build()
        }
    }
}
```

---

## Compose使用

```kotlin
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val repository: SettingsRepository
) : ViewModel() {
    val settings = repository.settings
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), AppSettings.getDefaultInstance())

    fun updateTheme(theme: String) = viewModelScope.launch { repository.updateTheme(theme) }
    fun setNotifications(enabled: Boolean) = viewModelScope.launch { repository.setNotifications(enabled) }
}

@Composable
fun ProtoSettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        SwitchSetting(
            title = "通知", checked = settings.notificationsEnabled,
            onCheckedChange = { viewModel.setNotifications(it) }
        )
        Text("テーマ: ${settings.theme.ifEmpty { "デフォルト" }}")
        Text("フォントサイズ: ${settings.fontSize}sp")
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| Proto DataStore | 型安全な設定保存 |
| `Serializer` | シリアライズ定義 |
| `updateData` | アトミック更新 |
| Protocol Buffers | スキーマ定義 |

- Protocol Buffersで型安全な設定スキーマ
- `Serializer`でバイナリ⇔オブジェクト変換
- `updateData`でアトミックな更新
- Preferences DataStoreより型安全

---

8種類のAndroidアプリテンプレート（設定管理対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
- [DataStore Migration](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-migration-2026)
- [設定画面](https://zenn.dev/myougatheaxo/articles/android-compose-compose-settings-screen-2026)
