---
title: "Compose チャートライブラリ Vico — 棒グラフ・折れ線・円グラフの実装"
emoji: "📊"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "chart"]
published: true
---

## この記事で学べること

Compose向けチャートライブラリ**Vico**を使って、棒グラフ・折れ線グラフを簡単に実装する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.patrykandpatrick.vico:compose-m3:2.0.0-beta.1")
}
```

---

## 折れ線グラフ

```kotlin
@Composable
fun LineChartExample() {
    val modelProducer = remember { CartesianChartModelProducer() }

    LaunchedEffect(Unit) {
        modelProducer.runTransaction {
            lineSeries {
                series(4, 12, 8, 16, 12, 20, 14)
            }
        }
    }

    CartesianChartHost(
        chart = rememberCartesianChart(
            rememberLineCartesianLayer()
        ),
        modelProducer = modelProducer,
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
    )
}
```

---

## 棒グラフ

```kotlin
@Composable
fun BarChartExample() {
    val modelProducer = remember { CartesianChartModelProducer() }

    LaunchedEffect(Unit) {
        modelProducer.runTransaction {
            columnSeries {
                series(5, 8, 3, 12, 7, 15, 9)
            }
        }
    }

    CartesianChartHost(
        chart = rememberCartesianChart(
            rememberColumnCartesianLayer()
        ),
        modelProducer = modelProducer,
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
    )
}
```

---

## 複合チャート（棒+線）

```kotlin
@Composable
fun CompositeChart() {
    val modelProducer = remember { CartesianChartModelProducer() }

    LaunchedEffect(Unit) {
        modelProducer.runTransaction {
            columnSeries {
                series(3, 7, 5, 9, 6, 11, 8)
            }
            lineSeries {
                series(4, 6, 7, 8, 7, 10, 9)
            }
        }
    }

    CartesianChartHost(
        chart = rememberCartesianChart(
            rememberColumnCartesianLayer(),
            rememberLineCartesianLayer()
        ),
        modelProducer = modelProducer,
        modifier = Modifier
            .fillMaxWidth()
            .height(250.dp)
    )
}
```

---

## ViewModelとの統合

```kotlin
class StatsViewModel(private val repository: StatsRepository) : ViewModel() {
    val modelProducer = CartesianChartModelProducer()

    init {
        viewModelScope.launch {
            repository.getWeeklyStats().collect { stats ->
                modelProducer.runTransaction {
                    lineSeries {
                        series(stats.map { it.value })
                    }
                }
            }
        }
    }
}

@Composable
fun StatsScreen(viewModel: StatsViewModel) {
    CartesianChartHost(
        chart = rememberCartesianChart(
            rememberLineCartesianLayer()
        ),
        modelProducer = viewModel.modelProducer,
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .padding(16.dp)
    )
}
```

---

## Canvasで自作する場合

```kotlin
@Composable
fun SimplePieChart(slices: List<Pair<Float, Color>>) {
    Canvas(modifier = Modifier.size(200.dp)) {
        val total = slices.sumOf { it.first.toDouble() }.toFloat()
        var startAngle = -90f
        slices.forEach { (value, color) ->
            val sweep = (value / total) * 360f
            drawArc(color, startAngle, sweep, useCenter = true)
            startAngle += sweep
        }
    }
}
```

円グラフはVicoでは非対応のため、Canvas自作が必要。

---

## まとめ

- **Vico**で折れ線・棒グラフを簡単に実装
- `CartesianChartModelProducer`でデータ更新
- `rememberLineCartesianLayer` / `rememberColumnCartesianLayer`
- 複合チャートも簡単に構成可能
- 円グラフはCanvas自作で対応

---

8種類のAndroidアプリテンプレート（チャート追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [ViewModel完全ガイド](https://zenn.dev/myougatheaxo/articles/android-viewmodel-stateflow-2026)
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
