---
title: "Proto DataStore完全ガイド — Protocol Buffers/型安全設定/マイグレーション"
emoji: "💾"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "datastore"]
published: true
---

## この記事で学べること

**Proto DataStore**（Protocol Buffers定義、型安全な読み書き、マイグレーション、Hilt統合）を解説します。

---

## Proto定義

```protobuf
// app/src/main/proto/settings.proto
syntax = "proto3";

option java_package = "com.example.app";
option java_multiple_files = true;

message AppSettings {
  ThemeMode theme_mode = 1;
  string language = 2;
  int32 font_size = 3;
  bool notifications_enabled = 4;
  NotificationSettings notification_settings = 5;
}

enum ThemeMode {
  SYSTEM = 0;
  LIGHT = 1;
  DARK = 2;
}

message NotificationSettings {
  bool sound = 1;
  bool vibration = 2;
  bool badge = 3;
}
```

---

## Serializer/DataStore

```kotlin
object AppSettingsSerializer : Serializer<AppSettings> {
    override val defaultValue: AppSettings = AppSettings.getDefaultInstance().toBuilder()
        .setThemeMode(ThemeMode.SYSTEM)
        .setLanguage("ja")
        .setFontSize(16)
        .setNotificationsEnabled(true)
        .setNotificationSettings(
            NotificationSettings.newBuilder()
                .setSound(true)
                .setVibration(true)
                .setBadge(true)
                .build()
        )
        .build()

    override suspend fun readFrom(input: InputStream): AppSettings {
        return try {
            AppSettings.parseFrom(input)
        } catch (e: InvalidProtocolBufferException) {
            defaultValue
        }
    }

    override suspend fun writeTo(t: AppSettings, output: OutputStream) {
        t.writeTo(output)
    }
}

// Hilt Module
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    @Provides
    @Singleton
    fun provideSettingsDataStore(@ApplicationContext context: Context): DataStore<AppSettings> {
        return DataStoreFactory.create(
            serializer = AppSettingsSerializer,
            produceFile = { context.dataStoreFile("app_settings.pb") }
        )
    }
}
```

---

## 読み書き

```kotlin
class SettingsRepository @Inject constructor(
    private val dataStore: DataStore<AppSettings>
) {
    val settings: Flow<AppSettings> = dataStore.data

    suspend fun setThemeMode(mode: ThemeMode) {
        dataStore.updateData { current ->
            current.toBuilder().setThemeMode(mode).build()
        }
    }

    suspend fun setFontSize(size: Int) {
        dataStore.updateData { current ->
            current.toBuilder().setFontSize(size).build()
        }
    }

    suspend fun toggleNotifications(enabled: Boolean) {
        dataStore.updateData { current ->
            current.toBuilder().setNotificationsEnabled(enabled).build()
        }
    }
}

// Compose
@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val settings by viewModel.settings.collectAsStateWithLifecycle(AppSettings.getDefaultInstance())

    Column(Modifier.padding(16.dp)) {
        Text("フォントサイズ: ${settings.fontSize}sp")
        Slider(
            value = settings.fontSize.toFloat(),
            onValueChange = { viewModel.setFontSize(it.toInt()) },
            valueRange = 12f..24f,
            steps = 5
        )
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| スキーマ | `.proto` ファイル |
| Serializer | `Serializer<T>` |
| 読み取り | `dataStore.data` Flow |
| 書き込み | `updateData { }` |

- Protocol Buffersで型安全な設定管理
- `Serializer`でデフォルト値を定義
- `updateData`でアトミックな更新
- Preferences DataStoreより型安全で構造化

---

8種類のAndroidアプリテンプレート（DataStore対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore Preferences](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [動的テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-dynamic-theme-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-dependency-injection-2026)
