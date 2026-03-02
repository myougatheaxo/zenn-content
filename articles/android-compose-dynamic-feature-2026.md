---
title: "Dynamic Feature Module ガイド — オンデマンド配信"
emoji: "📦"
type: "tech"
topics: ["android", "kotlin", "modular", "playstore"]
published: true
---

## この記事で学べること

**Dynamic Feature Module**（オンデマンドダウンロード、条件付き配信、Play Feature Delivery）を解説します。

---

## セットアップ

```kotlin
// dynamic feature module の build.gradle.kts
plugins {
    id("com.android.dynamic-feature")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.feature.premium"
    compileSdk = 35
}

dependencies {
    implementation(project(":app"))
}

// app/build.gradle.kts
android {
    dynamicFeatures += setOf(":feature:premium")
}
```

---

## マニフェスト設定

```xml
<!-- feature/premium/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:dist="http://schemas.android.com/apk/distribution">

    <dist:module
        dist:instant="false"
        dist:title="@string/premium_feature_title">
        <dist:delivery>
            <!-- オンデマンド配信 -->
            <dist:on-demand />
        </dist:delivery>
        <dist:fusing dist:include="true" />
    </dist:module>
</manifest>
```

---

## モジュールインストール

```kotlin
class FeatureInstaller(private val context: Context) {

    private val splitInstallManager = SplitInstallManagerFactory.create(context)

    private val _installState = MutableStateFlow<InstallState>(InstallState.NotInstalled)
    val installState: StateFlow<InstallState> = _installState.asStateFlow()

    fun isInstalled(moduleName: String): Boolean {
        return splitInstallManager.installedModules.contains(moduleName)
    }

    fun install(moduleName: String) {
        if (isInstalled(moduleName)) {
            _installState.value = InstallState.Installed
            return
        }

        val request = SplitInstallRequest.newBuilder()
            .addModule(moduleName)
            .build()

        splitInstallManager.registerListener { state ->
            when (state.status()) {
                SplitInstallSessionStatus.DOWNLOADING -> {
                    val progress = state.bytesDownloaded().toFloat() / state.totalBytesToDownload()
                    _installState.value = InstallState.Downloading(progress)
                }
                SplitInstallSessionStatus.INSTALLED -> {
                    _installState.value = InstallState.Installed
                }
                SplitInstallSessionStatus.FAILED -> {
                    _installState.value = InstallState.Error("インストール失敗: ${state.errorCode()}")
                }
                SplitInstallSessionStatus.REQUIRES_USER_CONFIRMATION -> {
                    _installState.value = InstallState.RequiresConfirmation
                }
                else -> {}
            }
        }

        splitInstallManager.startInstall(request)
            .addOnFailureListener { e ->
                _installState.value = InstallState.Error(e.message ?: "Error")
            }
    }

    fun uninstall(moduleName: String) {
        splitInstallManager.deferredUninstall(listOf(moduleName))
    }
}

sealed interface InstallState {
    data object NotInstalled : InstallState
    data class Downloading(val progress: Float) : InstallState
    data object Installed : InstallState
    data object RequiresConfirmation : InstallState
    data class Error(val message: String) : InstallState
}
```

---

## Compose UI

```kotlin
@Composable
fun PremiumFeatureButton(
    installer: FeatureInstaller = remember { FeatureInstaller(LocalContext.current) }
) {
    val state by installer.installState.collectAsStateWithLifecycle()

    when (val s = state) {
        InstallState.NotInstalled -> {
            Button(onClick = { installer.install("premium") }) {
                Icon(Icons.Default.Download, null)
                Spacer(Modifier.width(8.dp))
                Text("プレミアム機能をダウンロード")
            }
        }
        is InstallState.Downloading -> {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                LinearProgressIndicator(
                    progress = { s.progress },
                    modifier = Modifier.fillMaxWidth()
                )
                Text("ダウンロード中... ${(s.progress * 100).toInt()}%")
            }
        }
        InstallState.Installed -> {
            Button(onClick = { /* プレミアム機能を起動 */ }) {
                Icon(Icons.Default.Star, null)
                Spacer(Modifier.width(8.dp))
                Text("プレミアム機能を開く")
            }
        }
        is InstallState.Error -> {
            Text(s.message, color = MaterialTheme.colorScheme.error)
            Button(onClick = { installer.install("premium") }) {
                Text("再試行")
            }
        }
        InstallState.RequiresConfirmation -> {
            Text("ユーザー確認が必要です")
        }
    }
}
```

---

## まとめ

- `<dist:on-demand />`でオンデマンド配信
- `SplitInstallManager`でモジュールのインストール/アンインストール
- ダウンロード進捗を`SplitInstallSessionStatus`で監視
- `installedModules`でインストール済みかチェック
- APKサイズ削減（必要な機能だけダウンロード）
- Google Play Feature Delivery経由で配信

---

8種類のAndroidアプリテンプレート（モジュール設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [マルチモジュール](https://zenn.dev/myougatheaxo/articles/android-compose-multi-module-architecture-2026)
- [App Bundle](https://zenn.dev/myougatheaxo/articles/android-compose-build-gradle-tips-2026)
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
