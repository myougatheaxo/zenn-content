---
title: "Macrobenchmark完全ガイド — 起動時間/フレーム計測/スクロール性能"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Macrobenchmark**（起動時間計測、フレームタイミング、スクロール性能、CI統合）を解説します。

---

## 基本設定

```kotlin
// benchmark/build.gradle.kts
plugins {
    id("com.android.test")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.benchmark"
    targetProjectPath = ":app"
    experimentalProperties["android.experimental.self-instrumenting"] = true
}

dependencies {
    implementation("androidx.benchmark:benchmark-macro-junit4:1.3.3")
}
```

---

## 起動時間計測

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun startupCold() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }

    @Test
    fun startupWarm() = benchmarkRule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.WARM
    ) {
        pressHome()
        startActivityAndWait()
    }
}
```

---

## フレーム計測

```kotlin
@Test
fun scrollPerformance() = benchmarkRule.measureRepeated(
    packageName = "com.example.app",
    metrics = listOf(FrameTimingMetric()),
    iterations = 5,
    startupMode = StartupMode.COLD
) {
    startActivityAndWait()

    val list = device.findObject(By.res("item_list"))
    list.setGestureMargin(device.displayWidth / 5)

    repeat(3) {
        list.fling(Direction.DOWN)
        device.waitForIdle()
    }
}

@Test
fun navigationPerformance() = benchmarkRule.measureRepeated(
    packageName = "com.example.app",
    metrics = listOf(FrameTimingMetric()),
    iterations = 5
) {
    startActivityAndWait()

    // タブ切替
    device.findObject(By.desc("設定")).click()
    device.waitForIdle()

    device.findObject(By.desc("ホーム")).click()
    device.waitForIdle()
}
```

---

## Baseline Profileとの連携

```kotlin
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() = rule.collect(
        packageName = "com.example.app"
    ) {
        startActivityAndWait()

        // 主要画面を巡回
        device.findObject(By.text("一覧")).click()
        device.waitForIdle()

        val list = device.findObject(By.res("item_list"))
        list.fling(Direction.DOWN)
        device.waitForIdle()

        device.findObject(By.res("item_list")).children[0].click()
        device.waitForIdle()
    }
}
```

---

## まとめ

| メトリクス | 計測対象 |
|-----------|---------|
| StartupTimingMetric | 起動時間 |
| FrameTimingMetric | フレーム遅延 |
| TraceSectionMetric | カスタムトレース |

- `MacrobenchmarkRule`で実機ベンチマーク
- Cold/Warm/Hotスタートアップ計測
- `FrameTimingMetric`でジャンク検出
- Baseline Profile生成で起動高速化

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-baseline-profile-2026)
- [起動最適化](https://zenn.dev/myougatheaxo/articles/android-compose-startup-optimization-2026)
- [Compose Compiler Report](https://zenn.dev/myougatheaxo/articles/android-compose-compose-compiler-report-2026)
