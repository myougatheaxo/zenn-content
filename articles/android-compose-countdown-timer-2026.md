---
title: "カウントダウンタイマー実装ガイド — Composeで時間表示UIを作る"
emoji: "⏱️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "timer"]
published: true
---

## この記事で学べること

Composeで**カウントダウンタイマー**と**ストップウォッチ**を実装する方法を解説します。

---

## 基本のカウントダウン

```kotlin
@Composable
fun CountdownTimer(totalSeconds: Int) {
    var timeLeft by remember { mutableIntStateOf(totalSeconds) }
    var isRunning by remember { mutableStateOf(false) }

    LaunchedEffect(isRunning) {
        while (isRunning && timeLeft > 0) {
            delay(1000)
            timeLeft--
        }
        if (timeLeft == 0) isRunning = false
    }

    Column(
        Modifier.fillMaxWidth(),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            formatTime(timeLeft),
            style = MaterialTheme.typography.displayLarge,
            fontFamily = FontFamily.Monospace
        )

        Spacer(Modifier.height(24.dp))

        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            Button(onClick = { isRunning = !isRunning }) {
                Text(if (isRunning) "停止" else "開始")
            }
            OutlinedButton(onClick = {
                isRunning = false
                timeLeft = totalSeconds
            }) {
                Text("リセット")
            }
        }
    }
}

fun formatTime(totalSeconds: Int): String {
    val minutes = totalSeconds / 60
    val seconds = totalSeconds % 60
    return "%02d:%02d".format(minutes, seconds)
}
```

---

## 円形プログレス付きタイマー

```kotlin
@Composable
fun CircularCountdown(totalSeconds: Int) {
    var timeLeft by remember { mutableIntStateOf(totalSeconds) }
    var isRunning by remember { mutableStateOf(false) }
    val progress by animateFloatAsState(
        targetValue = timeLeft.toFloat() / totalSeconds,
        animationSpec = tween(300)
    )

    LaunchedEffect(isRunning) {
        while (isRunning && timeLeft > 0) {
            delay(1000)
            timeLeft--
        }
    }

    Box(contentAlignment = Alignment.Center) {
        Canvas(Modifier.size(200.dp)) {
            drawArc(
                color = Color.LightGray.copy(alpha = 0.3f),
                startAngle = 0f, sweepAngle = 360f,
                useCenter = false,
                style = Stroke(width = 16f, cap = StrokeCap.Round)
            )
            drawArc(
                color = Color(0xFF6200EE),
                startAngle = -90f,
                sweepAngle = progress * 360f,
                useCenter = false,
                style = Stroke(width = 16f, cap = StrokeCap.Round)
            )
        }

        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(
                formatTime(timeLeft),
                style = MaterialTheme.typography.headlineLarge,
                fontFamily = FontFamily.Monospace
            )
        }
    }
}
```

---

## ストップウォッチ

```kotlin
@Composable
fun Stopwatch() {
    var elapsedMs by remember { mutableLongStateOf(0L) }
    var isRunning by remember { mutableStateOf(false) }
    var laps by remember { mutableStateOf(listOf<Long>()) }

    LaunchedEffect(isRunning) {
        val startTime = System.currentTimeMillis() - elapsedMs
        while (isRunning) {
            elapsedMs = System.currentTimeMillis() - startTime
            delay(10)
        }
    }

    Column(
        Modifier.fillMaxWidth().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            formatStopwatch(elapsedMs),
            style = MaterialTheme.typography.displayMedium,
            fontFamily = FontFamily.Monospace
        )

        Spacer(Modifier.height(24.dp))

        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            Button(onClick = { isRunning = !isRunning }) {
                Text(if (isRunning) "停止" else "開始")
            }
            if (isRunning) {
                OutlinedButton(onClick = {
                    laps = laps + elapsedMs
                }) {
                    Text("ラップ")
                }
            } else if (elapsedMs > 0) {
                OutlinedButton(onClick = {
                    elapsedMs = 0
                    laps = emptyList()
                }) {
                    Text("リセット")
                }
            }
        }

        if (laps.isNotEmpty()) {
            Spacer(Modifier.height(16.dp))
            LazyColumn {
                itemsIndexed(laps.reversed()) { index, lap ->
                    ListItem(
                        headlineContent = { Text("ラップ ${laps.size - index}") },
                        trailingContent = { Text(formatStopwatch(lap)) }
                    )
                }
            }
        }
    }
}

fun formatStopwatch(ms: Long): String {
    val minutes = (ms / 60000) % 60
    val seconds = (ms / 1000) % 60
    val centis = (ms / 10) % 100
    return "%02d:%02d.%02d".format(minutes, seconds, centis)
}
```

---

## まとめ

- `LaunchedEffect` + `delay`でカウントダウン
- `Canvas` + `drawArc`で円形プログレス表示
- `System.currentTimeMillis()`ベースのストップウォッチ
- ラップ機能は`List<Long>`で記録
- `FontFamily.Monospace`で桁ズレ防止

---

8種類のAndroidアプリテンプレート（タイマー機能追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [AlarmManager完全ガイド](https://zenn.dev/myougatheaxo/articles/android-alarm-scheduler-2026)
- [Canvas完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-canvas-drawing-2026)
- [Composeアニメーション入門](https://zenn.dev/myougatheaxo/articles/compose-animation-basics-2026)
