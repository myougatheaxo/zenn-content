---
title: "Compose KMP完全ガイド — Kotlin Multiplatform/共有UI/expect-actual"
emoji: "🌍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "kmp"]
published: true
---

## この記事で学べること

**Compose KMP**（Kotlin Multiplatform、Compose Multiplatform、expect/actual、プラットフォーム分岐）を解説します。

---

## プロジェクト構成

```
shared/
├── src/
│   ├── commonMain/          # 共通コード
│   │   └── kotlin/
│   │       ├── App.kt       # 共通UI
│   │       └── Platform.kt  # expect宣言
│   ├── androidMain/         # Android固有
│   │   └── kotlin/
│   │       └── Platform.android.kt
│   └── iosMain/             # iOS固有
│       └── kotlin/
│           └── Platform.ios.kt
androidApp/                  # Androidアプリ
iosApp/                      # iOSアプリ
```

```kotlin
// build.gradle.kts (shared)
kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        commonMain.dependencies {
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.material3)
            implementation(compose.ui)
        }
    }
}
```

---

## expect/actual

```kotlin
// commonMain: expect宣言
expect fun getPlatformName(): String

expect class PlatformContext

expect fun PlatformContext.openUrl(url: String)

// androidMain: actual実装
actual fun getPlatformName(): String = "Android ${Build.VERSION.SDK_INT}"

actual typealias PlatformContext = Context

actual fun PlatformContext.openUrl(url: String) {
    startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(url)))
}

// iosMain: actual実装
actual fun getPlatformName(): String = "iOS ${UIDevice.currentDevice.systemVersion}"
```

---

## 共通Compose UI

```kotlin
// commonMain
@Composable
fun App() {
    MaterialTheme {
        var showDetails by remember { mutableStateOf(false) }

        Column(
            Modifier.fillMaxSize().padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text("Hello, ${getPlatformName()}!",
                style = MaterialTheme.typography.headlineMedium)
            Spacer(Modifier.height(16.dp))

            Button(onClick = { showDetails = !showDetails }) {
                Text("詳細を表示")
            }

            AnimatedVisibility(visible = showDetails) {
                ElevatedCard(Modifier.padding(top = 16.dp)) {
                    Column(Modifier.padding(16.dp)) {
                        Text("Kotlin Multiplatformで")
                        Text("Android/iOSの共通UIを構築")
                    }
                }
            }
        }
    }
}
```

---

## まとめ

| 概念 | 用途 |
|------|------|
| `commonMain` | 共通ロジック+UI |
| `expect/actual` | プラットフォーム分岐 |
| Compose Multiplatform | 共通UI |
| `androidMain`/`iosMain` | 固有実装 |

- Compose MultiplatformでAndroid/iOS共通UIを構築
- `expect/actual`でプラットフォーム固有処理を分離
- ビジネスロジック(ViewModel/Repository)も共有可能
- Android固有APIは`androidMain`で実装

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Desktop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-desktop-2026)
- [Compose MultiModule](https://zenn.dev/myougatheaxo/articles/android-compose-compose-multi-module-2026)
- [Compose WASM](https://zenn.dev/myougatheaxo/articles/android-compose-compose-wasm-2026)
