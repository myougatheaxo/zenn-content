---
title: "Proto DataStoreガイド — 型安全な設定管理"
emoji: "💾"
type: "tech"
topics: ["android", "kotlin", "datastore", "protobuf"]
published: true
---

## この記事で学べること

**Proto DataStore**で型安全にアプリ設定を管理する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
plugins {
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("androidx.datastore:datastore:1.1.0")
    implementation("com.google.protobuf:protobuf-javalite:4.28.0")
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:4.28.0"
    }
    generateProtoTasks {
        all().forEach { task ->
            task.builtins {
                register("java") { option("lite") }
            }
        }
    }
}
```

---

## Protoファイル定義

```protobuf
// src/main/proto/settings.proto
syntax = "proto3";

option java_package = "com.example.app";
option java_multiple_files = true;

message AppSettings {
    bool dark_mode = 1;
    string language = 2;
    int32 font_size = 3;
    bool notifications_enabled = 4;
    Theme theme = 5;
}

enum Theme {
    THEME_SYSTEM = 0;
    THEME_LIGHT = 1;
    THEME_DARK = 2;
}
```

---

## Serializer

```kotlin
object AppSettingsSerializer : Serializer<AppSettings> {
    override val defaultValue: AppSettings = AppSettings.getDefaultInstance()

    override suspend fun readFrom(input: InputStream): AppSettings {
        try {
            return AppSettings.parseFrom(input)
        } catch (e: InvalidProtocolBufferException) {
            throw CorruptionException("Cannot read proto.", e)
        }
    }

    override suspend fun writeTo(t: AppSettings, output: OutputStream) {
        t.writeTo(output)
    }
}
```

---

## DataStore作成とViewModel

```kotlin
// DataStore定義
val Context.settingsDataStore: DataStore<AppSettings> by dataStore(
    fileName = "settings.pb",
    serializer = AppSettingsSerializer
)

// ViewModel
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val dataStore: DataStore<AppSettings>
) : ViewModel() {

    val settings = dataStore.data
        .stateIn(viewModelScope, SharingStarted.Eagerly, AppSettings.getDefaultInstance())

    fun setDarkMode(enabled: Boolean) {
        viewModelScope.launch {
            dataStore.updateData { current ->
                current.toBuilder()
                    .setDarkMode(enabled)
                    .build()
            }
        }
    }

    fun setFontSize(size: Int) {
        viewModelScope.launch {
            dataStore.updateData { current ->
                current.toBuilder()
                    .setFontSize(size)
                    .build()
            }
        }
    }

    fun setTheme(theme: Theme) {
        viewModelScope.launch {
            dataStore.updateData { current ->
                current.toBuilder()
                    .setTheme(theme)
                    .build()
            }
        }
    }
}
```

---

## Compose画面

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val settings by viewModel.settings.collectAsStateWithLifecycle()

    LazyColumn {
        item {
            SwitchPreference(
                title = "ダークモード",
                checked = settings.darkMode,
                onCheckedChange = { viewModel.setDarkMode(it) }
            )
        }
        item {
            SwitchPreference(
                title = "通知",
                checked = settings.notificationsEnabled,
                onCheckedChange = { viewModel.setNotifications(it) }
            )
        }
    }
}
```

---

## まとめ

- Proto DataStoreは型安全（Preferences DataStoreより堅牢）
- `.proto`ファイルでスキーマ定義
- `Serializer`でシリアライズ/デシリアライズ
- `updateData { it.toBuilder().set...().build() }`で更新
- `dataStore.data`をFlowとしてCompose側で監視
- スキーマ変更時はフィールド番号を変えない（後方互換）

---

8種類のAndroidアプリテンプレート（設定管理実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore設定管理](https://zenn.dev/myougatheaxo/articles/android-datastore-2026)
- [テーマ切り替え](https://zenn.dev/myougatheaxo/articles/android-compose-theme-switcher-2026)
- [Hilt依存性注入](https://zenn.dev/myougatheaxo/articles/android-hilt-dependency-injection-2026)
