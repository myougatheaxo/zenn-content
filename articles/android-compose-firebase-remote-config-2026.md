---
title: "Firebase Remote Config完全ガイド — A/Bテスト/機能フラグ/Compose連携"
emoji: "🎛️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Firebase Remote Config**（リアルタイム更新、機能フラグ、A/Bテスト、Compose連携）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
    implementation("com.google.firebase:firebase-config-ktx")
}
```

---

## デフォルト値設定

```xml
<!-- res/xml/remote_config_defaults.xml -->
<?xml version="1.0" encoding="utf-8"?>
<defaultsMap>
    <entry>
        <key>feature_new_ui</key>
        <value>false</value>
    </entry>
    <entry>
        <key>welcome_message</key>
        <value>Welcome!</value>
    </entry>
    <entry>
        <key>max_items</key>
        <value>20</value>
    </entry>
</defaultsMap>
```

---

## Repository実装

```kotlin
class RemoteConfigRepository @Inject constructor() {
    private val remoteConfig = Firebase.remoteConfig.apply {
        setConfigSettingsAsync(remoteConfigSettings {
            minimumFetchIntervalInSeconds = if (BuildConfig.DEBUG) 0 else 3600
        })
        setDefaultsAsync(R.xml.remote_config_defaults)
    }

    private val _configUpdates = MutableSharedFlow<Unit>(replay = 1)

    init {
        remoteConfig.addOnConfigUpdateListener(object : ConfigUpdateListener {
            override fun onUpdate(configUpdate: ConfigUpdate) {
                remoteConfig.activate().addOnCompleteListener {
                    _configUpdates.tryEmit(Unit)
                }
            }
            override fun onError(error: FirebaseRemoteConfigException) {
                Log.e("RemoteConfig", "Update error", error)
            }
        })
        fetchAndActivate()
    }

    private fun fetchAndActivate() {
        remoteConfig.fetchAndActivate()
    }

    fun getBoolean(key: String): Boolean = remoteConfig.getBoolean(key)
    fun getString(key: String): String = remoteConfig.getString(key)
    fun getLong(key: String): Long = remoteConfig.getLong(key)

    val updates: SharedFlow<Unit> = _configUpdates
}
```

---

## 機能フラグ

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val remoteConfig: RemoteConfigRepository
) : ViewModel() {

    val isNewUiEnabled: StateFlow<Boolean> = remoteConfig.updates
        .map { remoteConfig.getBoolean("feature_new_ui") }
        .stateIn(viewModelScope, SharingStarted.Eagerly, remoteConfig.getBoolean("feature_new_ui"))

    val welcomeMessage: StateFlow<String> = remoteConfig.updates
        .map { remoteConfig.getString("welcome_message") }
        .stateIn(viewModelScope, SharingStarted.Eagerly, remoteConfig.getString("welcome_message"))
}
```

---

## Compose画面

```kotlin
@Composable
fun HomeScreen(viewModel: MainViewModel = hiltViewModel()) {
    val isNewUi by viewModel.isNewUiEnabled.collectAsStateWithLifecycle()
    val welcomeMessage by viewModel.welcomeMessage.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        Text(welcomeMessage, style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        if (isNewUi) {
            NewFeatureCard()
        } else {
            LegacyContent()
        }
    }
}

@Composable
fun NewFeatureCard() {
    Card(Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp)) {
            Text("新機能", style = MaterialTheme.typography.titleMedium)
            Text("Remote Configで有効化された新しいUIです")
        }
    }
}
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| 初期化 | `Firebase.remoteConfig` |
| デフォルト値 | `setDefaultsAsync()` |
| リアルタイム | `addOnConfigUpdateListener` |
| 機能フラグ | `getBoolean("key")` |
| A/Bテスト | Firebase Console設定 |

- `fetchAndActivate()`で最新値取得
- `ConfigUpdateListener`でリアルタイム反映
- デフォルト値XMLでオフライン時も動作
- Firebase ConsoleでA/Bテスト設定

---

8種類のAndroidアプリテンプレート（Firebase連携対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
- [Firebase Firestore](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-firestore-2026)
- [Firebase Messaging](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-messaging-2026)
