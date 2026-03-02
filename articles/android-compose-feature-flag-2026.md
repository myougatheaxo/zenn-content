---
title: "Feature Flag実装完全ガイド — ローカル/Remote Config/段階的ロールアウト"
emoji: "🚩"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "featureflag"]
published: true
---

## この記事で学べること

**Feature Flag**（ローカルフラグ、Remote Config連携、段階的ロールアウト、A/Bテスト、Compose UI切替）を解説します。

---

## FeatureFlagManager

```kotlin
interface FeatureFlagManager {
    fun isEnabled(flag: FeatureFlag): Boolean
    fun getVariant(flag: FeatureFlag): String
}

enum class FeatureFlag(val key: String, val defaultValue: Boolean) {
    NEW_HOME_UI("new_home_ui", false),
    DARK_MODE_V2("dark_mode_v2", false),
    PREMIUM_FEATURES("premium_features", false),
    AI_ASSISTANT("ai_assistant", false)
}

class FeatureFlagManagerImpl @Inject constructor(
    private val remoteConfig: RemoteConfigRepository,
    private val localOverrides: DataStore<Preferences>
) : FeatureFlagManager {

    override fun isEnabled(flag: FeatureFlag): Boolean {
        // ローカルオーバーライドを優先（デバッグ用）
        val localOverride = runBlocking {
            localOverrides.data.first()[booleanPreferencesKey(flag.key)]
        }
        if (localOverride != null) return localOverride

        // Remote Config
        return remoteConfig.getBoolean(flag.key)
    }

    override fun getVariant(flag: FeatureFlag): String {
        return remoteConfig.getString("${flag.key}_variant")
    }
}
```

---

## Compose連携

```kotlin
@Composable
fun FeatureGate(
    flag: FeatureFlag,
    featureFlagManager: FeatureFlagManager = hiltViewModel<SettingsViewModel>().featureFlagManager,
    enabledContent: @Composable () -> Unit,
    disabledContent: @Composable () -> Unit = {}
) {
    if (featureFlagManager.isEnabled(flag)) {
        enabledContent()
    } else {
        disabledContent()
    }
}

// 使用
@Composable
fun HomeScreen() {
    FeatureGate(
        flag = FeatureFlag.NEW_HOME_UI,
        enabledContent = { NewHomeContent() },
        disabledContent = { LegacyHomeContent() }
    )
}
```

---

## デバッグメニュー

```kotlin
@Composable
fun FeatureFlagDebugScreen(viewModel: DebugViewModel = hiltViewModel()) {
    val overrides by viewModel.overrides.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        Text("Feature Flags", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        FeatureFlag.entries.forEach { flag ->
            Row(
                Modifier.fillMaxWidth().padding(vertical = 8.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Column(Modifier.weight(1f)) {
                    Text(flag.name, style = MaterialTheme.typography.titleSmall)
                    Text(flag.key, style = MaterialTheme.typography.bodySmall)
                }
                Switch(
                    checked = overrides[flag.key] ?: flag.defaultValue,
                    onCheckedChange = { viewModel.setOverride(flag, it) }
                )
            }
        }

        Spacer(Modifier.height(16.dp))
        OutlinedButton(onClick = { viewModel.clearAllOverrides() }) {
            Text("全てリセット")
        }
    }
}
```

---

## まとめ

| 手法 | 用途 |
|------|------|
| ローカルフラグ | デバッグ/開発中 |
| Remote Config | 本番ロールアウト |
| A/Bテスト | バリアント比較 |
| デバッグメニュー | QAテスト |

- Feature Flagで安全に新機能をリリース
- Remote Configで段階的ロールアウト
- ローカルオーバーライドでデバッグ時に強制切替
- `FeatureGate` ComposableでUI分岐を宣言的に

---

8種類のAndroidアプリテンプレート（Feature Flag対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Firebase Remote Config](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-remote-config-2026)
- [Firebase Analytics](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-analytics-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
