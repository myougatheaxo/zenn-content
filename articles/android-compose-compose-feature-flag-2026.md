---
title: "Compose Feature Flag完全ガイド — Firebase RemoteConfig/A-Bテスト/段階ロールアウト"
emoji: "🚩"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "firebase"]
published: true
---

## この記事で学べること

**Compose Feature Flag**（Firebase Remote Config、A/Bテスト、段階的ロールアウト、フラグ管理）を解説します。

---

## Remote Config設定

```groovy
// build.gradle
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.1.0"))
    implementation("com.google.firebase:firebase-config-ktx")
}
```

```kotlin
class FeatureFlagManager @Inject constructor() {
    private val remoteConfig = Firebase.remoteConfig.apply {
        setConfigSettingsAsync(remoteConfigSettings {
            minimumFetchIntervalInSeconds = if (BuildConfig.DEBUG) 0 else 3600
        })
        setDefaultsAsync(mapOf(
            "new_checkout_enabled" to false,
            "max_items_per_page" to 20L,
            "welcome_message" to "ようこそ！"
        ))
    }

    suspend fun fetch() {
        remoteConfig.fetchAndActivate().await()
    }

    fun isEnabled(flag: String): Boolean = remoteConfig.getBoolean(flag)
    fun getLong(key: String): Long = remoteConfig.getLong(key)
    fun getString(key: String): String = remoteConfig.getString(key)
}
```

---

## Compose連携

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val featureFlags: FeatureFlagManager
) : ViewModel() {
    val newCheckoutEnabled = MutableStateFlow(false)
    val welcomeMessage = MutableStateFlow("")

    init {
        viewModelScope.launch {
            featureFlags.fetch()
            newCheckoutEnabled.value = featureFlags.isEnabled("new_checkout_enabled")
            welcomeMessage.value = featureFlags.getString("welcome_message")
        }
    }
}

@Composable
fun HomeScreen(viewModel: HomeViewModel = hiltViewModel()) {
    val newCheckout by viewModel.newCheckoutEnabled.collectAsStateWithLifecycle()
    val welcome by viewModel.welcomeMessage.collectAsStateWithLifecycle()

    Column(Modifier.padding(16.dp)) {
        Text(welcome, style = MaterialTheme.typography.headlineMedium)

        if (newCheckout) {
            // 新しいチェックアウトUI
            Button(onClick = { /* 新フロー */ }) { Text("新しいチェックアウト") }
        } else {
            // 既存のチェックアウトUI
            Button(onClick = { /* 既存フロー */ }) { Text("購入する") }
        }
    }
}
```

---

## ローカルフラグ（開発用）

```kotlin
object LocalFeatureFlags {
    private val overrides = mutableMapOf<String, Boolean>()

    fun setOverride(flag: String, enabled: Boolean) { overrides[flag] = enabled }
    fun clearOverrides() { overrides.clear() }
    fun getOverride(flag: String): Boolean? = overrides[flag]
}

// Debug画面
@Composable
fun FeatureFlagDebugScreen() {
    val flags = listOf("new_checkout_enabled", "dark_mode_v2", "ai_recommendations")

    LazyColumn(Modifier.padding(16.dp)) {
        items(flags) { flag ->
            var enabled by remember { mutableStateOf(LocalFeatureFlags.getOverride(flag) ?: false) }
            ListItem(
                headlineContent = { Text(flag) },
                trailingContent = {
                    Switch(checked = enabled, onCheckedChange = {
                        enabled = it
                        LocalFeatureFlags.setOverride(flag, it)
                    })
                }
            )
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| Remote Config | サーバーサイドフラグ |
| `fetchAndActivate` | 設定取得+適用 |
| `setDefaults` | デフォルト値設定 |
| A/Bテスト | Firebase Console設定 |

- Firebase Remote Configでサーバーから動的にフラグ制御
- `minimumFetchIntervalInSeconds`で取得頻度を制御
- デフォルト値設定でオフライン時も安全に動作
- デバッグ画面でローカルオーバーライド可能に

---

8種類のAndroidアプリテンプレート（Feature Flag対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose FirebaseAnalytics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-analytics-2026)
- [Compose FirebaseAuth](https://zenn.dev/myougatheaxo/articles/android-compose-compose-firebase-auth-2026)
- [Hilt Multibinding](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-multibinding-2026)
