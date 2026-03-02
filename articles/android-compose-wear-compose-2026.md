---
title: "Wear OS Compose完全ガイド — ScalingLazyColumn/TimeText/CurvedLayout"
emoji: "⌚"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "wearos"]
published: true
---

## この記事で学べること

**Wear OS Compose**（ScalingLazyColumn、TimeText、CurvedLayout、丸型画面対応）を解説します。

---

## 基本構造

```kotlin
@Composable
fun WearApp() {
    MaterialTheme {
        Scaffold(
            timeText = { TimeText() },
            vignette = { Vignette(vignettePosition = VignettePosition.TopAndBottom) },
            positionIndicator = { PositionIndicator(scalingLazyListState = listState) }
        ) {
            val listState = rememberScalingLazyListState()

            ScalingLazyColumn(
                state = listState,
                modifier = Modifier.fillMaxSize(),
                anchorType = ScalingLazyListAnchorType.ItemCenter
            ) {
                item { ListHeader { Text("設定") } }

                item {
                    Chip(
                        onClick = {},
                        label = { Text("通知") },
                        icon = { Icon(Icons.Default.Notifications, null) },
                        modifier = Modifier.fillMaxWidth()
                    )
                }

                item {
                    ToggleChip(
                        checked = true,
                        onCheckedChange = {},
                        label = { Text("バイブレーション") },
                        toggleControl = { Switch(checked = true) },
                        modifier = Modifier.fillMaxWidth()
                    )
                }
            }
        }
    }
}
```

---

## CurvedLayout

```kotlin
@Composable
fun CurvedTextExample() {
    CurvedLayout(modifier = Modifier.fillMaxSize()) {
        curvedRow {
            curvedText(
                text = "Wear OS App",
                style = CurvedTextStyle(
                    fontSize = 16.sp,
                    color = MaterialTheme.colors.primary
                )
            )
        }
    }
}

// 丸型画面対応
@Composable
fun RoundScreenContent() {
    val isRound = LocalConfiguration.current.isScreenRound

    Box(
        modifier = Modifier
            .fillMaxSize()
            .let { if (isRound) it.padding(20.dp) else it.padding(8.dp) },
        contentAlignment = Alignment.Center
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text("12:34", style = MaterialTheme.typography.display1)
            Text("歩数: 5,432", style = MaterialTheme.typography.body1)
        }
    }
}
```

---

## DataLayer通信

```kotlin
class WearDataClient @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val dataClient = Wearable.getDataClient(context)
    private val messageClient = Wearable.getMessageClient(context)

    suspend fun sendMessage(path: String, data: ByteArray) {
        val nodes = Wearable.getNodeClient(context).connectedNodes.await()
        nodes.forEach { node ->
            messageClient.sendMessage(node.id, path, data).await()
        }
    }

    fun observeData(path: String): Flow<DataMap> = callbackFlow {
        val listener = DataClient.OnDataChangedListener { dataEvents ->
            dataEvents.forEach { event ->
                if (event.type == DataEvent.TYPE_CHANGED && event.dataItem.uri.path == path) {
                    trySend(DataMapItem.fromDataItem(event.dataItem).dataMap)
                }
            }
        }
        dataClient.addListener(listener)
        awaitClose { dataClient.removeListener(listener) }
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|---------------|------|
| ScalingLazyColumn | リスト表示 |
| CurvedLayout | 曲面テキスト |
| TimeText | 時刻表示 |
| Chip/ToggleChip | ボタン |

- `ScalingLazyColumn`で丸型画面に最適化されたリスト
- `CurvedLayout`で画面端に沿ったテキスト
- `Scaffold`で時刻、ビネット、スクロールインジケータ
- DataLayerでスマホ↔Watch間通信

---

8種類のAndroidアプリテンプレート（Wear OS対応可）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Health Connect](https://zenn.dev/myougatheaxo/articles/android-compose-health-connect-2026)
- [センサー](https://zenn.dev/myougatheaxo/articles/android-compose-sensor-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
