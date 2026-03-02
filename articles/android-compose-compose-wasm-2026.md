---
title: "Compose WASM完全ガイド — Kotlin/Wasm/ブラウザ向けCompose UI"
emoji: "🕸️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "wasm"]
published: true
---

## この記事で学べること

**Compose WASM**（Kotlin/Wasm、Compose for Web、ブラウザ向けUI、WASMセットアップ）を解説します。

---

## プロジェクトセットアップ

```kotlin
// build.gradle.kts
plugins {
    kotlin("multiplatform")
    id("org.jetbrains.compose")
    id("org.jetbrains.kotlin.plugin.compose")
}

kotlin {
    wasmJs {
        browser()
        binaries.executable()
    }

    sourceSets {
        val wasmJsMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material3)
                implementation(compose.ui)
            }
        }
    }
}
```

---

## エントリーポイント

```kotlin
// src/wasmJsMain/kotlin/Main.kt
import androidx.compose.ui.ExperimentalComposeUiApi
import androidx.compose.ui.window.CanvasBasedWindow

@OptIn(ExperimentalComposeUiApi::class)
fun main() {
    CanvasBasedWindow(canvasElementId = "ComposeTarget") {
        App()
    }
}

// index.html
// <canvas id="ComposeTarget"></canvas>
// <script src="your-app.js"></script>
```

---

## 共通UI

```kotlin
@Composable
fun App() {
    MaterialTheme {
        var counter by remember { mutableIntStateOf(0) }

        Column(
            Modifier.fillMaxSize().padding(32.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Text("Compose for WASM", style = MaterialTheme.typography.headlineLarge)
            Spacer(Modifier.height(16.dp))
            Text("カウント: $counter", style = MaterialTheme.typography.displayMedium)
            Spacer(Modifier.height(16.dp))
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                Button(onClick = { counter-- }) { Text("-") }
                Button(onClick = { counter++ }) { Text("+") }
                OutlinedButton(onClick = { counter = 0 }) { Text("リセット") }
            }
        }
    }
}
```

---

## 実行

```bash
# 開発サーバー起動
./gradlew wasmJsBrowserDevelopmentRun

# プロダクションビルド
./gradlew wasmJsBrowserDistribution
# 出力: build/dist/wasmJs/productionExecutable/
```

---

## まとめ

| 概念 | 用途 |
|------|------|
| Kotlin/Wasm | WASMコンパイル |
| `CanvasBasedWindow` | Canvas描画 |
| Compose Multiplatform | 共有UI |
| `wasmJsBrowserRun` | 開発サーバー |

- Kotlin/WasmでCompose UIをブラウザで実行
- Canvas要素にCompose UIを描画
- Android/iOS/Desktop/Webで同じUIコードを共有
- プロダクションビルドは静的ファイルとしてデプロイ

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose KMP](https://zenn.dev/myougatheaxo/articles/android-compose-compose-kmp-2026)
- [Compose Desktop](https://zenn.dev/myougatheaxo/articles/android-compose-compose-desktop-2026)
- [Compose MultiModule](https://zenn.dev/myougatheaxo/articles/android-compose-compose-multi-module-2026)
