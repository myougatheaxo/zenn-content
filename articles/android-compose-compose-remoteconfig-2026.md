---
title: "Compose RemoteConfig完全ガイド — Firebase設定/リアルタイム更新/条件付き配信/デフォルト値"
emoji: "☁️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose RemoteConfig**（Firebase Remote Config、リアルタイム更新、条件付き配信、Compose状態連携）を解説します。

---

## 基本設定

```kotlin
@HiltViewModel
class ConfigViewModel @Inject constructor() : ViewModel() {
    private val remoteConfig = Firebase.remoteConfig

    val bannerText = MutableStateFlow("")
    val maxItems = MutableStateFlow(10)
    val themeColor = MutableStateFlow("#6200EE")

    init {
        remoteConfig.setConfigSettingsAsync(remoteConfigSettings {
            minimumFetchIntervalInSeconds = if (BuildConfig.DEBUG) 0 else 3600
        })
        remoteConfig.setDefaultsAsync(R.xml.remote_config_defaults)

        viewModelScope.launch {
            remoteConfig.fetchAndActivate().await()
            updateValues()
        }
    }

    private fun updateValues() {
        bannerText.value = remoteConfig.getString("banner_text")
        maxItems.value = remoteConfig.getLong("max_items").toInt()
        themeColor.value = remoteConfig.getString("theme_color")
    }
}
```

---

## リアルタイム更新

```kotlin
@HiltViewModel
class RealTimeConfigViewModel @Inject constructor() : ViewModel() {
    private val remoteConfig = Firebase.remoteConfig
    val maintenanceMode = MutableStateFlow(false)
    val appVersion = MutableStateFlow("")

    init {
        // リアルタイムリスナー
        remoteConfig.addOnConfigUpdateListener(object : ConfigUpdateListener {
            override fun onUpdate(configUpdate: ConfigUpdate) {
                remoteConfig.activate().addOnCompleteListener {
                    maintenanceMode.value = remoteConfig.getBoolean("maintenance_mode")
                    appVersion.value = remoteConfig.getString("min_app_version")
                }
            }
            override fun onError(error: FirebaseRemoteConfigException) {
                Log.e("RemoteConfig", "Update error", error)
            }
        })
    }
}

@Composable
fun MainScreen(viewModel: RealTimeConfigViewModel = hiltViewModel()) {
    val maintenance by viewModel.maintenanceMode.collectAsStateWithLifecycle()

    if (maintenance) {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Icon(Icons.Default.Build, contentDescription = null, modifier = Modifier.size(64.dp))
                Spacer(Modifier.height(16.dp))
                Text("メンテナンス中です", style = MaterialTheme.typography.headlineMedium)
            }
        }
    } else {
        // 通常のUI
        Text("アプリコンテンツ")
    }
}
```

---

## 条件付きUI

```kotlin
@Composable
fun ConditionalUI(viewModel: ConfigViewModel = hiltViewModel()) {
    val banner by viewModel.bannerText.collectAsStateWithLifecycle()
    val maxItems by viewModel.maxItems.collectAsStateWithLifecycle()
    val color by viewModel.themeColor.collectAsStateWithLifecycle()

    val themeColor = remember(color) {
        try { Color(android.graphics.Color.parseColor(color)) }
        catch (e: Exception) { Color(0xFF6200EE) }
    }

    Column(Modifier.padding(16.dp)) {
        if (banner.isNotEmpty()) {
            Card(
                colors = CardDefaults.cardColors(containerColor = themeColor),
                modifier = Modifier.fillMaxWidth()
            ) {
                Text(banner, Modifier.padding(16.dp), color = Color.White)
            }
            Spacer(Modifier.height(16.dp))
        }

        LazyColumn {
            items(maxItems) { index ->
                ListItem(headlineContent = { Text("アイテム ${index + 1}") })
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `fetchAndActivate` | 設定取得+有効化 |
| `addOnConfigUpdateListener` | リアルタイム更新 |
| `setDefaults` | デフォルト値 |
| `minimumFetchInterval` | 取得間隔制御 |

- `fetchAndActivate()`で設定をサーバーから取得して適用
- リアルタイムリスナーで変更を即座に反映
- デフォルト値設定でオフライン時も安全動作
- Firebase Consoleで条件付き配信（国、バージョン等）

---

8種類のAndroidアプリテンプレート（Firebase対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FeatureFlag](https://zenn.dev/myougatheaxo/articles/android-compose-compose-feature-flag-2026)
- [Compose FirebaseAnalytics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-analytics-2026)
- [Compose FirebaseCrashlytics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-crashlytics-2026)
