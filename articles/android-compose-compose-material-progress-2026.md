---
title: "Compose Progress完全ガイド — LinearProgress/CircularProgress/カスタムインジケーター"
emoji: "⏳"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Compose Progress**（LinearProgressIndicator、CircularProgressIndicator、確定/不確定、カスタムアニメーション）を解説します。

---

## 基本プログレス

```kotlin
@Composable
fun ProgressDemo() {
    var progress by remember { mutableFloatStateOf(0.3f) }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(24.dp)) {
        // 確定（Determinate）
        Text("ダウンロード: ${(progress * 100).toInt()}%")
        LinearProgressIndicator(progress = { progress }, Modifier.fillMaxWidth())

        // 不確定（Indeterminate）
        Text("読み込み中...")
        LinearProgressIndicator(Modifier.fillMaxWidth())

        Button(onClick = { progress = (progress + 0.1f).coerceAtMost(1f) }) {
            Text("進捗+10%")
        }
    }
}
```

---

## CircularProgressIndicator

```kotlin
@Composable
fun CircularProgressDemo() {
    var loading by remember { mutableStateOf(true) }

    Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)) {

        // 不確定
        if (loading) {
            CircularProgressIndicator()
        }

        // 確定 + サイズ変更
        CircularProgressIndicator(
            progress = { 0.7f },
            modifier = Modifier.size(64.dp),
            strokeWidth = 6.dp,
            color = MaterialTheme.colorScheme.tertiary,
            trackColor = MaterialTheme.colorScheme.tertiaryContainer
        )

        Button(onClick = { loading = !loading }) {
            Text(if (loading) "停止" else "開始")
        }
    }
}
```

---

## アニメーション付きプログレス

```kotlin
@Composable
fun AnimatedProgressBar() {
    var targetProgress by remember { mutableFloatStateOf(0f) }
    val animatedProgress by animateFloatAsState(
        targetValue = targetProgress,
        animationSpec = tween(durationMillis = 1000, easing = FastOutSlowInEasing),
        label = "progress"
    )

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        LinearProgressIndicator(
            progress = { animatedProgress },
            modifier = Modifier.fillMaxWidth().height(8.dp),
            color = MaterialTheme.colorScheme.primary,
            trackColor = MaterialTheme.colorScheme.surfaceVariant
        )
        Text("${(animatedProgress * 100).toInt()}%", style = MaterialTheme.typography.titleLarge)

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { targetProgress = (targetProgress + 0.25f).coerceAtMost(1f) }) {
                Text("+25%")
            }
            OutlinedButton(onClick = { targetProgress = 0f }) { Text("リセット") }
        }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `LinearProgressIndicator` | 横バー型プログレス |
| `CircularProgressIndicator` | 円形プログレス |
| `progress` パラメータ | 確定/不確定の切替 |
| `animateFloatAsState` | スムーズアニメーション |

- `progress`パラメータ省略で不確定（ローディング）表示
- `progress`にラムダを渡すと確定プログレス
- `strokeWidth`/`color`/`trackColor`でカスタマイズ
- `animateFloatAsState`でスムーズな進捗アニメーション

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Slider](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-slider-2026)
- [Compose Switch](https://zenn.dev/myougatheaxo/articles/android-compose-compose-material-switch-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
