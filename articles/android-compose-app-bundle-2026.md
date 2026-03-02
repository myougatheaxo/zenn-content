---
title: "App Bundle完全ガイド — AAB/動的配信/Play Feature Delivery/サイズ最適化"
emoji: "📦"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "playstore"]
published: true
---

## この記事で学べること

**App Bundle**（AABビルド、動的配信、Play Feature Delivery、サイズ最適化）を解説します。

---

## AABビルド設定

```kotlin
// build.gradle.kts (app)
android {
    bundle {
        language {
            enableSplit = true  // 言語別分割
        }
        density {
            enableSplit = true  // 画面密度別分割
        }
        abi {
            enableSplit = true  // ABI別分割
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}
```

---

## Play Feature Delivery

```kotlin
// 動的機能モジュール
// dynamic_feature/build.gradle.kts
plugins {
    id("com.android.dynamic-feature")
}

android {
    namespace = "com.example.app.camera"
}

dependencies {
    implementation(project(":app"))
}

// 動的モジュールのインストール
class DynamicFeatureManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val splitInstallManager = SplitInstallManagerFactory.create(context)

    fun installModule(moduleName: String): Flow<InstallState> = callbackFlow {
        val request = SplitInstallRequest.newBuilder()
            .addModule(moduleName)
            .build()

        val listener = SplitInstallStateUpdatedListener { state ->
            trySend(InstallState(
                status = state.status(),
                bytesDownloaded = state.bytesDownloaded(),
                totalBytes = state.totalBytesToDownload()
            ))
        }

        splitInstallManager.registerListener(listener)
        splitInstallManager.startInstall(request)

        awaitClose { splitInstallManager.unregisterListener(listener) }
    }

    fun isInstalled(moduleName: String): Boolean {
        return splitInstallManager.installedModules.contains(moduleName)
    }
}

data class InstallState(val status: Int, val bytesDownloaded: Long, val totalBytes: Long)
```

---

## Compose UI

```kotlin
@Composable
fun DynamicFeatureButton(
    moduleName: String,
    label: String,
    viewModel: FeatureViewModel = hiltViewModel()
) {
    val installed by viewModel.isInstalled(moduleName).collectAsStateWithLifecycle(false)
    val installState by viewModel.installState.collectAsStateWithLifecycle()

    if (installed) {
        Button(onClick = { /* 機能を起動 */ }) {
            Text(label)
        }
    } else {
        OutlinedButton(onClick = { viewModel.installModule(moduleName) }) {
            Icon(Icons.Default.Download, null)
            Spacer(Modifier.width(8.dp))
            Text("$label をダウンロード")
        }
        installState?.let { state ->
            if (state.totalBytes > 0) {
                LinearProgressIndicator(
                    progress = { state.bytesDownloaded.toFloat() / state.totalBytes },
                    modifier = Modifier.fillMaxWidth()
                )
            }
        }
    }
}
```

---

## まとめ

| 機能 | 効果 |
|------|------|
| Language Split | 不要言語を除外 |
| Density Split | 不要解像度を除外 |
| ABI Split | 不要CPUアーキ除外 |
| Dynamic Feature | オンデマンド配信 |

- AABでGoogle Playが最適なAPKを自動生成
- `enableSplit`で言語/密度/ABIを分割配信
- Play Feature Deliveryでオンデマンドモジュール
- `isMinifyEnabled`+`isShrinkResources`でサイズ削減

---

8種類のAndroidアプリテンプレート（AAB最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ProGuard/R8](https://zenn.dev/myougatheaxo/articles/android-compose-proguard-r8-2026)
- [Google Play公開](https://zenn.dev/myougatheaxo/articles/android-compose-google-play-release-2026)
- [Gradle Cache](https://zenn.dev/myougatheaxo/articles/android-compose-gradle-cache-2026)
