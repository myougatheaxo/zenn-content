---
title: "Wear OS Compose完全ガイド — ScalingLazyColumn/Chip/TimeText"
emoji: "⌚"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "wearos"]
published: true
---

## この記事で学べること

**Wear OS Compose**（ScalingLazyColumn、Chip、TimeText、丸型レイアウト）を解説します。

---

## ScalingLazyColumn

```kotlin
@Composable
fun WearHomeScreen() {
    val listState = rememberScalingLazyListState()

    Scaffold(
        timeText = { TimeText() },
        vignette = { Vignette(vignettePosition = VignettePosition.TopAndBottom) },
        positionIndicator = { PositionIndicator(scalingLazyListState = listState) }
    ) {
        ScalingLazyColumn(
            state = listState,
            modifier = Modifier.fillMaxSize(),
            anchorType = ScalingLazyListAnchorType.ItemCenter
        ) {
            item {
                Text("タスク一覧", style = MaterialTheme.typography.title3, textAlign = TextAlign.Center)
            }
            items(5) { index ->
                Chip(
                    onClick = {},
                    label = { Text("タスク ${index + 1}") },
                    icon = { Icon(Icons.Default.Check, null, Modifier.size(ChipDefaults.IconSize)) },
                    modifier = Modifier.fillMaxWidth()
                )
            }
        }
    }
}
```

---

## Chip/ToggleChip

```kotlin
@Composable
fun WearSettingsScreen() {
    var notificationsEnabled by remember { mutableStateOf(true) }
    var vibrationEnabled by remember { mutableStateOf(true) }

    ScalingLazyColumn(Modifier.fillMaxSize()) {
        item { ListHeader { Text("設定") } }

        item {
            ToggleChip(
                checked = notificationsEnabled,
                onCheckedChange = { notificationsEnabled = it },
                label = { Text("通知") },
                toggleControl = { Switch(checked = notificationsEnabled) },
                modifier = Modifier.fillMaxWidth()
            )
        }

        item {
            ToggleChip(
                checked = vibrationEnabled,
                onCheckedChange = { vibrationEnabled = it },
                label = { Text("バイブレーション") },
                toggleControl = { Switch(checked = vibrationEnabled) },
                modifier = Modifier.fillMaxWidth()
            )
        }

        item {
            CompactChip(
                onClick = { /* リセット */ },
                label = { Text("設定をリセット") }
            )
        }
    }
}
```

---

## 確認ダイアログ

```kotlin
@Composable
fun WearConfirmation() {
    var showConfirm by remember { mutableStateOf(false) }

    Button(onClick = { showConfirm = true }, Modifier.fillMaxWidth()) {
        Text("完了")
    }

    if (showConfirm) {
        Confirmation(
            onTimeout = { showConfirm = false },
            icon = { Icon(Icons.Default.Check, null) },
            content = { Text("完了しました") }
        )
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ScalingLazyColumn` | スケーリング付きリスト |
| `Chip` | タップ可能アイテム |
| `ToggleChip` | ON/OFF切替 |
| `TimeText` | 時刻表示 |

- `ScalingLazyColumn`で丸型画面に最適化
- `Chip`/`ToggleChip`でWear OS標準UI
- `Vignette`で画面端のグラデーション
- `TimeText`でステータスバー時刻

---

8種類のAndroidアプリテンプレート（マルチデバイス対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Adaptive Layout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
- [NavigationRail](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-rail-2026)
- [Glance Widget](https://zenn.dev/myougatheaxo/articles/android-compose-compose-widget-glance-2026)
