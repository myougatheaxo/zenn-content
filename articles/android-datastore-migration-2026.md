---
title: "SharedPreferences卒業ガイド — DataStoreへの移行を3ステップで解説"
emoji: "💾"
type: "tech"
topics: ["android", "kotlin", "datastore", "jetpackcompose"]
published: true
---

## この記事で学べること

Android開発で設定値を保存するとき、昔は`SharedPreferences`が定番でした。しかし2026年現在、**Googleは公式にDataStoreへの移行を推奨**しています。

AIが生成するアプリも既にDataStoreを採用しています。この記事では「なぜDataStoreなのか」と「移行の3ステップ」を解説します。

---

## SharedPreferencesの問題

| 問題 | 影響 |
|------|------|
| UIスレッドでI/O | ANR（アプリ無反応）の原因 |
| 型安全性なし | `getString`で取得した値をIntにキャストして落ちる |
| エラーハンドリング不可 | 書き込み失敗を検知できない |
| 変更監視が煩雑 | `OnSharedPreferenceChangeListener`が不安定 |

---

## DataStoreの2種類

### Preferences DataStore

キーバリュー形式。SharedPreferencesの直接の後継。

```kotlin
// 依存関係
implementation("androidx.datastore:datastore-preferences:1.1.1")
```

```kotlin
val Context.dataStore by preferencesDataStore(name = "settings")

// キーの定義
object PreferencesKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val USER_NAME = stringPreferencesKey("user_name")
    val NOTIFICATION_INTERVAL = intPreferencesKey("notification_interval")
}
```

### Proto DataStore

型付きオブジェクト形式。Protocol Buffersを使う。複雑な構造にはこちら。

個人アプリなら**Preferences DataStoreで十分**です。

---

## 移行の3ステップ

### Step 1: DataStoreインスタンスの作成

```kotlin
// DataStoreModule.kt
val Context.dataStore by preferencesDataStore(name = "settings")
```

ファイルのトップレベルに1行書くだけ。

### Step 2: 読み取り（Flow）

```kotlin
class SettingsRepository(private val dataStore: DataStore<Preferences>) {

    val darkMode: Flow<Boolean> = dataStore.data
        .map { preferences ->
            preferences[PreferencesKeys.DARK_MODE] ?: false
        }

    val userName: Flow<String> = dataStore.data
        .map { preferences ->
            preferences[PreferencesKeys.USER_NAME] ?: "User"
        }
}
```

`Flow`で返ってくるので、Composeの`collectAsState()`でそのまま使えます。

### Step 3: 書き込み（suspend）

```kotlin
suspend fun setDarkMode(enabled: Boolean) {
    dataStore.edit { preferences ->
        preferences[PreferencesKeys.DARK_MODE] = enabled
    }
}

suspend fun setUserName(name: String) {
    dataStore.edit { preferences ->
        preferences[PreferencesKeys.USER_NAME] = name
    }
}
```

`edit`はsuspend関数なので、`viewModelScope.launch`内で呼びます。

---

## ViewModelでの使い方

```kotlin
class SettingsViewModel(
    private val repository: SettingsRepository
) : ViewModel() {

    val darkMode = repository.darkMode
        .stateIn(viewModelScope, SharingStarted.Lazily, false)

    fun toggleDarkMode() {
        viewModelScope.launch {
            val current = darkMode.value
            repository.setDarkMode(!current)
        }
    }
}
```

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel) {
    val isDark by viewModel.darkMode.collectAsState()

    Row(verticalAlignment = Alignment.CenterVertically) {
        Text("ダークモード")
        Spacer(Modifier.weight(1f))
        Switch(
            checked = isDark,
            onCheckedChange = { viewModel.toggleDarkMode() }
        )
    }
}
```

---

## SharedPreferencesからの自動マイグレーション

既存アプリを移行する場合、DataStoreにはマイグレーション機能があります。

```kotlin
val Context.dataStore by preferencesDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(SharedPreferencesMigration(context, "old_shared_prefs"))
    }
)
```

これを指定するだけで、初回起動時に旧SharedPreferencesのデータが自動的にDataStoreに移行されます。

---

## AIが生成するDataStoreコード

Claude Codeでアプリを生成すると、最初からDataStoreが採用されます。

| 特徴 | AIの実装 |
|------|---------|
| 種類 | Preferences DataStore |
| キー定義 | companion objectまたはobject |
| 読み取り | Flow + stateIn |
| 書き込み | suspend + edit |
| DI | 手動注入（Hilt不使用） |

小〜中規模アプリではHiltなしの手動注入で十分。AIはこの判断を正しく行っています。

---

## まとめ

- SharedPreferencesは非推奨。UIスレッドブロック・型安全性なし
- **Preferences DataStore**がシンプルで推奨
- 移行は3ステップ：インスタンス作成 → Flow読み取り → suspend書き込み
- AIが生成するコードは最初からDataStoreを採用済み

---

8種類のAndroidアプリテンプレート（全てDataStore採用・SharedPreferences不使用）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AIでAndroidアプリを47秒で作った話](https://zenn.dev/myougatheaxo/articles/ai-android-app-47seconds)
- [ViewModel + StateFlow入門](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Kotlin Coroutine入門](https://zenn.dev/myougatheaxo/articles/kotlin-coroutine-android-basics)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
