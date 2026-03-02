---
title: "Compose Dynamic Feature完全ガイド — オンデマンド配信/モジュール分割/SplitInstall"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "dynamicfeature"]
published: true
---

## この記事で学べること

**Compose Dynamic Feature**（オンデマンド配信、SplitInstallManager、モジュール分割、ダウンロード状態UI）を解説します。

---

## Dynamic Featureモジュール

```groovy
// dynamic_feature/build.gradle
plugins {
    id("com.android.dynamic-feature")
}

android {
    namespace = "com.example.app.scanner"
}

dependencies {
    implementation(project(":app"))
}
```

```xml
<!-- dynamic_feature/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:dist="http://schemas.android.com/apk/distribution">

    <dist:module
        dist:instant="false"
        dist:title="@string/title_scanner">
        <dist:delivery>
            <dist:on-demand />
        </dist:delivery>
        <dist:fusing dist:include="true" />
    </dist:module>
</manifest>
```

---

## SplitInstall管理

```kotlin
class DynamicFeatureManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val splitInstallManager = SplitInstallManagerFactory.create(context)

    val installState = MutableStateFlow<InstallState>(InstallState.NotInstalled)

    fun isInstalled(moduleName: String): Boolean =
        splitInstallManager.installedModules.contains(moduleName)

    fun install(moduleName: String) {
        val request = SplitInstallRequest.newBuilder()
            .addModule(moduleName)
            .build()

        installState.value = InstallState.Downloading(0)

        splitInstallManager.registerListener { state ->
            when (state.status()) {
                SplitInstallSessionStatus.DOWNLOADING -> {
                    val progress = (state.bytesDownloaded() * 100 / state.totalBytesToDownload()).toInt()
                    installState.value = InstallState.Downloading(progress)
                }
                SplitInstallSessionStatus.INSTALLED -> {
                    installState.value = InstallState.Installed
                }
                SplitInstallSessionStatus.FAILED -> {
                    installState.value = InstallState.Error("インストール失敗")
                }
                else -> {}
            }
        }

        splitInstallManager.startInstall(request)
    }
}

sealed class InstallState {
    data object NotInstalled : InstallState()
    data class Downloading(val progress: Int) : InstallState()
    data object Installed : InstallState()
    data class Error(val message: String) : InstallState()
}
```

---

## Compose UI

```kotlin
@Composable
fun DynamicFeatureScreen(manager: DynamicFeatureManager) {
    val state by manager.installState.collectAsStateWithLifecycle()

    Column(Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally) {

        when (val s = state) {
            is InstallState.NotInstalled -> {
                Text("スキャナー機能", style = MaterialTheme.typography.headlineMedium)
                Spacer(Modifier.height(16.dp))
                Button(onClick = { manager.install("scanner") }) {
                    Text("ダウンロードして使う")
                }
            }
            is InstallState.Downloading -> {
                Text("ダウンロード中...", style = MaterialTheme.typography.titleMedium)
                Spacer(Modifier.height(16.dp))
                LinearProgressIndicator(progress = { s.progress / 100f },
                    modifier = Modifier.fillMaxWidth())
                Text("${s.progress}%")
            }
            is InstallState.Installed -> {
                Text("インストール完了!", style = MaterialTheme.typography.titleMedium)
                Button(onClick = { /* navigate to scanner */ }) { Text("スキャナーを開く") }
            }
            is InstallState.Error -> {
                Text(s.message, color = MaterialTheme.colorScheme.error)
                Button(onClick = { manager.install("scanner") }) { Text("再試行") }
            }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `SplitInstallManager` | モジュール管理 |
| `SplitInstallRequest` | インストール要求 |
| `on-demand` | オンデマンド配信 |
| `SessionStatus` | 状態監視 |

- Dynamic Feature Moduleで機能を分割・オンデマンド配信
- `SplitInstallManager`でダウンロード/インストールを管理
- ダウンロード進捗をCompose UIにリアルタイム反映
- APKサイズ削減とユーザー体験の最適化

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose MultiModule](https://zenn.dev/myougatheaxo/articles/android-compose-compose-multi-module-2026)
- [Compose R8Optimization](https://zenn.dev/myougatheaxo/articles/android-compose-compose-r8-optimization-2026)
- [Compose BaselineProfile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
