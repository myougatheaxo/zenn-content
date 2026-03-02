---
title: "DataStore Migration完全ガイド — SharedPreferences移行/バージョン管理"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "datastore"]
published: true
---

## この記事で学べること

**DataStore Migration**（SharedPreferencesからの移行、バージョン管理、データ変換）を解説します。

---

## SharedPreferencesからの移行

```kotlin
val Context.dataStore by preferencesDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(SharedPreferencesMigration(context, "old_prefs"))
    }
)

// 移行元のSharedPreferencesキーをDataStoreキーにマッピング
val Context.dataStoreWithMapping by preferencesDataStore(
    name = "settings_v2",
    produceMigrations = { context ->
        listOf(
            SharedPreferencesMigration(
                context = context,
                sharedPreferencesName = "old_prefs",
                keysToMigrate = setOf("dark_mode", "font_size", "language")
            )
        )
    }
)
```

---

## Proto DataStore Migration

```kotlin
val Context.settingsStore by dataStore(
    fileName = "settings.pb",
    serializer = AppSettingsSerializer,
    produceMigrations = { context ->
        listOf(
            SharedPreferencesMigration(context, "old_prefs") { prefs, currentData ->
                currentData.toBuilder()
                    .setTheme(prefs.getString("theme", "light") ?: "light")
                    .setNotificationsEnabled(prefs.getBoolean("notifications", true))
                    .setFontSize(prefs.getInt("font_size", 16))
                    .build()
            }
        )
    }
)
```

---

## カスタムMigration

```kotlin
class SettingsMigration(private val context: Context) : DataMigration<Preferences> {
    override suspend fun shouldMigrate(currentData: Preferences): Boolean {
        return currentData[intPreferencesKey("schema_version")] != 2
    }

    override suspend fun migrate(currentData: Preferences): Preferences {
        val mutable = currentData.toMutablePreferences()
        // v1→v2: キー名変更
        mutable[stringPreferencesKey("display_name")]?.let { name ->
            mutable[stringPreferencesKey("user_name")] = name
            mutable.remove(stringPreferencesKey("display_name"))
        }
        mutable[intPreferencesKey("schema_version")] = 2
        return mutable.toPreferences()
    }

    override suspend fun cleanUp() {
        context.getSharedPreferences("old_prefs", Context.MODE_PRIVATE).edit().clear().apply()
    }
}
```

---

## まとめ

| Migration | 用途 |
|-----------|------|
| `SharedPreferencesMigration` | SP→DataStore |
| `DataMigration` | カスタム移行 |
| `keysToMigrate` | 移行キー指定 |
| `cleanUp` | 移行後クリーンアップ |

- `SharedPreferencesMigration`で自動移行
- `keysToMigrate`で特定キーのみ移行
- `DataMigration`でカスタム変換ロジック
- `cleanUp`で古いデータを削除

---

8種類のAndroidアプリテンプレート（DataStore対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
- [Proto DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-proto-2026)
- [Room Migration](https://zenn.dev/myougatheaxo/articles/android-compose-room-migration-advanced-2026)
