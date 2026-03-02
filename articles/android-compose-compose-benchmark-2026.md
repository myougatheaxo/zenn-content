---
title: "Benchmark完全ガイド — Macrobenchmark/Microbenchmark/スタートアップ計測"
emoji: "⏱️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Benchmark**（Macrobenchmark、Microbenchmark、スタートアップ時間計測、フレームタイミング）を解説します。

---

## Macrobenchmark

```kotlin
// :benchmark モジュール
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
@RunWith(AndroidJUnit4::class)
class ScrollBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

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
}
```

---

## Microbenchmark

```kotlin
@RunWith(AndroidJUnit4::class)
class SortBenchmark {
    @get:Rule
    val benchmarkRule = BenchmarkRule()

    @Test
    fun sortItems() = benchmarkRule.measureRepeated {
        val items = runWithTimingDisabled {
            (1..1000).map { Item(it, "Item $it", (Math.random() * 100).toInt()) }.shuffled()
        }
        items.sortedBy { it.priority }
    }

    @Test
    fun filterItems() = benchmarkRule.measureRepeated {
        val items = runWithTimingDisabled {
            (1..10000).map { Item(it, "Item $it", it % 5) }
        }
        items.filter { it.priority > 2 }.map { it.name }
    }
}
```

---

## まとめ

| ベンチマーク | 用途 |
|------------|------|
| `Macrobenchmark` | アプリ全体（起動/スクロール） |
| `Microbenchmark` | 関数/アルゴリズム単位 |
| `StartupTimingMetric` | 起動時間 |
| `FrameTimingMetric` | フレーム描画時間 |

- Macrobenchmarkでアプリ起動・スクロール性能計測
- Microbenchmarkで処理ロジックの性能計測
- Cold/Warm/Hotスタートアップの違いを計測
- CI/CDに組み込んでパフォーマンス劣化を検出

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Baseline Profile](https://zenn.dev/myougatheaxo/articles/android-compose-compose-baseline-profile-2026)
- [Screenshot Test](https://zenn.dev/myougatheaxo/articles/android-compose-compose-screenshot-test-2026)
- [Stable Annotation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-stable-annotation-2026)
