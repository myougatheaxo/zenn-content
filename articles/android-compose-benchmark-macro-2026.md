---
title: "Macrobenchmark完全ガイド — 起動時間/スクロール/Baseline Profile生成"
emoji: "🏎️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Macrobenchmark**（アプリ起動時間計測、スクロールパフォーマンス、Baseline Profile自動生成、CIでの計測）を解説します。

---

## セットアップ

```kotlin
// benchmark/build.gradle.kts
plugins {
    id("com.android.test")
    id("org.jetbrains.kotlin.android")
}
android {
    namespace = "com.example.benchmark"
    compileSdk = 35
    defaultConfig {
        minSdk = 28
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
    targetProjectPath = ":app"
    experimentalProperties["android.experimental.self-instrumenting"] = true
}
dependencies {
    implementation("androidx.benchmark:benchmark-macro-junit4:1.3.3")
}
```

---

## 起動時間ベンチマーク

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

## スクロールベンチマーク

```kotlin
@RunWith(AndroidJUnit4::class)
class ScrollBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun scrollList() = benchmarkRule.measureRepeated(
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
}
```

---

## Baseline Profile生成

```kotlin
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() = rule.collect(
        packageName = "com.example.app"
    ) {
        // 起動
        pressHome()
        startActivityAndWait()

        // 主要ユーザーフロー
        device.findObject(By.text("ホーム")).click()
        device.waitForIdle()

        // リストスクロール
        val list = device.findObject(By.res("item_list"))
        list?.fling(Direction.DOWN)
        device.waitForIdle()

        // 詳細画面遷移
        device.findObject(By.res("list_item"))?.click()
        device.waitForIdle()

        // 設定画面
        device.pressBack()
        device.findObject(By.text("設定")).click()
        device.waitForIdle()
    }
}
```

---

## CI連携

```yaml
# .github/workflows/benchmark.yml
name: Benchmark
on:
  pull_request:
    paths: ['app/**']
jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          arch: x86_64
          script: ./gradlew :benchmark:connectedCheck
      - uses: actions/upload-artifact@v4
        with:
          name: benchmark-results
          path: benchmark/build/outputs/connected_android_test_additional_output/
```

---

## まとめ

| メトリクス | 計測内容 |
|-----------|---------|
| `StartupTimingMetric` | 起動時間 |
| `FrameTimingMetric` | フレームタイミング |
| `TraceSectionMetric` | カスタム区間 |
| `BaselineProfileRule` | Profile生成 |

- `MacrobenchmarkRule`で実機ベンチマーク
- Cold/Warm/Hot起動の各計測
- `FrameTimingMetric`でジャンク検出
- `BaselineProfileRule`で自動Profile生成

---

8種類のAndroidアプリテンプレート（パフォーマンス計測済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-baseline-profile-2026)
- [安定性/再コンポジション](https://zenn.dev/myougatheaxo/articles/android-compose-stability-performance-2026)
- [LazyColumn最適化](https://zenn.dev/myougatheaxo/articles/android-compose-lazy-performance-2026)
