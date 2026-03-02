---
title: "Baseline Profile完全ガイド — 起動速度/スクロール最適化"
emoji: "⚡"
type: "tech"
topics: ["android", "kotlin", "performance", "baseline"]
published: true
---

## この記事で学べること

**Baseline Profile**（AOTコンパイル、起動速度改善、スクロール最適化、Macrobenchmark）を解説します。

---

## セットアップ

```kotlin
// settings.gradle.kts
include(":benchmark")

// benchmark/build.gradle.kts
plugins {
    id("com.android.test")
    id("org.jetbrains.kotlin.android")
    id("androidx.baselineprofile")
}

android {
    namespace = "com.example.benchmark"
    compileSdk = 35
    targetProjectPath = ":app"
}

baselineProfile {
    useConnectedDevices = true
}

dependencies {
    implementation("androidx.test.ext:junit:1.2.1")
    implementation("androidx.benchmark:benchmark-macro-junit4:1.3.3")
}

// app/build.gradle.kts
plugins {
    id("androidx.baselineprofile")
}

dependencies {
    baselineProfile(project(":benchmark"))
}
```

---

## Baseline Profile生成

```kotlin
// benchmark/src/main/java/BaselineProfileGenerator.kt
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {

    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateProfile() {
        rule.collect(
            packageName = "com.example.myapp",
            includeInStartupProfile = true
        ) {
            // アプリ起動
            pressHome()
            startActivityAndWait()

            // メイン画面操作
            device.waitForIdle()

            // スクロール
            device.findObject(By.scrollable(true))?.apply {
                repeat(3) {
                    scroll(Direction.DOWN, 1f)
                    device.waitForIdle()
                }
            }

            // タブ切替
            device.findObject(By.text("検索"))?.click()
            device.waitForIdle()

            device.findObject(By.text("プロフィール"))?.click()
            device.waitForIdle()
        }
    }
}
```

---

## Macrobenchmark（起動時間計測）

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val rule = MacrobenchmarkRule()

    @Test
    fun coldStartup() {
        rule.measureRepeated(
            packageName = "com.example.myapp",
            metrics = listOf(StartupTimingMetric()),
            startupMode = StartupMode.COLD,
            iterations = 10
        ) {
            pressHome()
            startActivityAndWait()
        }
    }

    @Test
    fun warmStartup() {
        rule.measureRepeated(
            packageName = "com.example.myapp",
            metrics = listOf(StartupTimingMetric()),
            startupMode = StartupMode.WARM,
            iterations = 10
        ) {
            pressHome()
            startActivityAndWait()
        }
    }
}
```

---

## スクロールベンチマーク

```kotlin
@RunWith(AndroidJUnit4::class)
class ScrollBenchmark {

    @get:Rule
    val rule = MacrobenchmarkRule()

    @Test
    fun scrollPerformance() {
        rule.measureRepeated(
            packageName = "com.example.myapp",
            metrics = listOf(FrameTimingMetric()),
            iterations = 5
        ) {
            startActivityAndWait()

            val list = device.findObject(By.scrollable(true))
            list?.apply {
                repeat(5) {
                    scroll(Direction.DOWN, 2f)
                    device.waitForIdle()
                }
            }
        }
    }
}
```

---

## Profile適用

```
# Baseline Profile生成
./gradlew :benchmark:pixel6Api34BenchmarkAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=BaselineProfileGenerator

# 生成されたプロファイルは自動的に
# app/src/main/baseline-prof.txt に配置

# Release ビルドでAOTコンパイルに使用される
./gradlew :app:assembleRelease
```

---

## 効果測定

```kotlin
// Before/After比較
// Cold Start: 800ms → 500ms（37%改善）
// Warm Start: 400ms → 250ms（37%改善）
// Frame Duration P99: 16ms → 8ms（50%改善）

// Baseline Profileの効果:
// - 起動時に必要なコードをAOTコンパイル
// - JITコンパイルのオーバーヘッド削減
// - スクロール時のジャンク軽減
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `BaselineProfileRule` | Profile生成 |
| `MacrobenchmarkRule` | パフォーマンス計測 |
| `StartupTimingMetric` | 起動時間計測 |
| `FrameTimingMetric` | フレーム時間計測 |
| `baseline-prof.txt` | AOTコンパイル対象リスト |

- Baseline Profileで起動速度30-40%改善
- Macrobenchmarkで客観的な計測
- Release ビルドで自動適用
- Google Playに自動配信（Cloud Profile）

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [App Startup](https://zenn.dev/myougatheaxo/articles/android-compose-app-startup-2026)
- [パフォーマンスプロファイリング](https://zenn.dev/myougatheaxo/articles/android-compose-performance-profiling-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-column-tips-2026)
