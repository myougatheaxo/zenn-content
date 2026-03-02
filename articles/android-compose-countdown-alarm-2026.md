---
title: "カウントダウン/タイマーガイド — CountDownTimer/AlarmManager連携"
emoji: "⏰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "timer"]
published: true
---

## この記事で学べること

Composeでの**カウントダウンタイマー**（CountDownTimer、LaunchedEffect、AlarmManager連携）を解説します。

---

## LaunchedEffectタイマー

```kotlin
@Composable
fun SimpleTimer() {
    var timeLeft by remember { mutableLongStateOf(60L) }
    var isRunning by remember { mutableStateOf(false) }

    LaunchedEffect(isRunning) {
        while (isRunning && timeLeft > 0) {
            delay(1000)
            timeLeft--
        }
        if (timeLeft == 0L) isRunning = false
    }

    Column(
        Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            formatTime(timeLeft),
            style = MaterialTheme.typography.displayLarge
        )
        Spacer(Modifier.height(16.dp))
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { isRunning = !isRunning }) {
                Text(if (isRunning) "一時停止" else "開始")
            }
            OutlinedButton(onClick = { timeLeft = 60; isRunning = false }) {
                Text("リセット")
            }
        }
    }
}

fun formatTime(seconds: Long): String {
    val min = seconds / 60
    val sec = seconds % 60
    return "%02d:%02d".format(min, sec)
}
```

---

## 円形プログレスタイマー

```kotlin
@Composable
fun CircularTimer(totalSeconds: Long = 300L) {
    var timeLeft by remember { mutableLongStateOf(totalSeconds) }
    var isRunning by remember { mutableStateOf(false) }
    val progress by animateFloatAsState(
        targetValue = timeLeft.toFloat() / totalSeconds,
        animationSpec = tween(1000, easing = LinearEasing),
        label = "progress"
    )

    LaunchedEffect(isRunning) {
        while (isRunning && timeLeft > 0) {
            delay(1000)
            timeLeft--
        }
    }

    Box(contentAlignment = Alignment.Center) {
        CircularProgressIndicator(
            progress = { progress },
            modifier = Modifier.size(200.dp),
            strokeWidth = 8.dp,
            trackColor = MaterialTheme.colorScheme.surfaceVariant
        )
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(
                formatTime(timeLeft),
                style = MaterialTheme.typography.headlineLarge
            )
            Text(
                if (isRunning) "実行中" else "停止",
                style = MaterialTheme.typography.bodySmall
            )
        }
    }
}
```

---

## ViewModelでのタイマー管理

```kotlin
class TimerViewModel : ViewModel() {

    private val _timeLeft = MutableStateFlow(0L)
    val timeLeft: StateFlow<Long> = _timeLeft.asStateFlow()

    private val _isRunning = MutableStateFlow(false)
    val isRunning: StateFlow<Boolean> = _isRunning.asStateFlow()

    private var timerJob: Job? = null

    fun start(seconds: Long) {
        _timeLeft.value = seconds
        _isRunning.value = true
        timerJob = viewModelScope.launch {
            while (_timeLeft.value > 0) {
                delay(1000)
                _timeLeft.value--
            }
            _isRunning.value = false
        }
    }

    fun pause() {
        timerJob?.cancel()
        _isRunning.value = false
    }

    fun resume() {
        _isRunning.value = true
        timerJob = viewModelScope.launch {
            while (_timeLeft.value > 0) {
                delay(1000)
                _timeLeft.value--
            }
            _isRunning.value = false
        }
    }

    fun reset() {
        timerJob?.cancel()
        _timeLeft.value = 0
        _isRunning.value = false
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
    val laps = remember { mutableStateListOf<Long>() }

    LaunchedEffect(isRunning) {
        while (isRunning) {
            delay(10)
            elapsedMs += 10
        }
    }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            formatMillis(elapsedMs),
            style = MaterialTheme.typography.displayMedium,
            fontFamily = FontFamily.Monospace
        )

        Spacer(Modifier.height(16.dp))

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { isRunning = !isRunning }) {
                Text(if (isRunning) "停止" else "開始")
            }
            OutlinedButton(onClick = {
                if (isRunning) laps.add(0, elapsedMs)
                else { elapsedMs = 0; laps.clear() }
            }) {
                Text(if (isRunning) "ラップ" else "リセット")
            }
        }

        LazyColumn(Modifier.weight(1f)) {
            itemsIndexed(laps) { index, lap ->
                ListItem(
                    headlineContent = { Text("Lap ${laps.size - index}") },
                    trailingContent = { Text(formatMillis(lap)) }
                )
            }
        }
    }
}

fun formatMillis(ms: Long): String {
    val min = (ms / 60000) % 60
    val sec = (ms / 1000) % 60
    val centis = (ms / 10) % 100
    return "%02d:%02d.%02d".format(min, sec, centis)
}
```

---

## まとめ

- `LaunchedEffect` + `delay`でシンプルなタイマー
- `CircularProgressIndicator`で円形プログレス表示
- ViewModelで`Job`管理（一時停止/再開/リセット）
- ストップウォッチはミリ秒精度で更新
- ラップタイム記録は`mutableStateListOf`
- `formatTime`/`formatMillis`でフォーマット

---

8種類のAndroidアプリテンプレート（タイマー機能設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [カウントダウンタイマー実装](https://zenn.dev/myougatheaxo/articles/android-compose-countdown-timer-2026)
- [AlarmManagerスケジューラー](https://zenn.dev/myougatheaxo/articles/android-alarm-scheduler-2026)
- [Foreground Serviceガイド](https://zenn.dev/myougatheaxo/articles/android-foreground-service-2026)
