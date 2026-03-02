---
title: "Picture-in-Picture完全ガイド — PiPモード/自動遷移/カスタムアクション"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "media"]
published: true
---

## この記事で学べること

**Picture-in-Picture（PiP）**（PiPモード起動、自動遷移、カスタムアクション、Compose対応）を解説します。

---

## PiP起動

```kotlin
class VideoActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            VideoPlayerWithPiP(
                onEnterPiP = { enterPipMode() }
            )
        }
    }

    private fun enterPipMode() {
        val params = PictureInPictureParams.Builder()
            .setAspectRatio(Rational(16, 9))
            .setActions(createPipActions())
            .build()

        enterPictureInPictureMode(params)
    }

    // ホームボタン押下時に自動PiP
    override fun onUserLeaveHint() {
        super.onUserLeaveHint()
        if (isVideoPlaying) {
            enterPipMode()
        }
    }

    private fun createPipActions(): List<RemoteAction> {
        val playIntent = PendingIntent.getBroadcast(
            this, 0,
            Intent("ACTION_PLAY_PAUSE"),
            PendingIntent.FLAG_IMMUTABLE
        )

        return listOf(
            RemoteAction(
                Icon.createWithResource(this, R.drawable.ic_play),
                "再生/一時停止", "再生/一時停止",
                playIntent
            )
        )
    }
}
```

---

## Compose PiP対応

```kotlin
@Composable
fun VideoPlayerWithPiP(onEnterPiP: () -> Unit) {
    val context = LocalContext.current
    val isInPipMode = rememberIsInPipMode()

    Column(Modifier.fillMaxSize()) {
        // 動画プレーヤー（常に表示）
        VideoPlayer(modifier = Modifier
            .fillMaxWidth()
            .aspectRatio(16f / 9f)
        )

        // PiP時はコントロールを非表示
        if (!isInPipMode) {
            Row(
                Modifier.fillMaxWidth().padding(16.dp),
                horizontalArrangement = Arrangement.SpaceEvenly
            ) {
                IconButton(onClick = {}) { Icon(Icons.Default.SkipPrevious, "前へ") }
                IconButton(onClick = {}) { Icon(Icons.Default.PlayArrow, "再生") }
                IconButton(onClick = {}) { Icon(Icons.Default.SkipNext, "次へ") }
                IconButton(onClick = onEnterPiP) { Icon(Icons.Default.PictureInPicture, "PiP") }
            }
        }
    }
}

@Composable
fun rememberIsInPipMode(): Boolean {
    val activity = LocalContext.current as Activity
    var isInPipMode by remember { mutableStateOf(activity.isInPictureInPictureMode) }

    DisposableEffect(activity) {
        val callback = object : ComponentActivity.PictureInPictureModeChangedCallback() {
            override fun onPictureInPictureModeChanged(isInPiP: Boolean) {
                isInPipMode = isInPiP
            }
        }
        (activity as ComponentActivity).addOnPictureInPictureModeChangedListener(callback)
        onDispose { activity.removeOnPictureInPictureModeChangedListener(callback) }
    }

    return isInPipMode
}
```

---

## まとめ

| 機能 | API |
|------|-----|
| PiP起動 | `enterPictureInPictureMode` |
| アスペクト比 | `setAspectRatio` |
| カスタムアクション | `RemoteAction` |
| 自動遷移 | `onUserLeaveHint` |

- `PictureInPictureParams`でPiPモード設定
- `onUserLeaveHint`でホームボタン時に自動PiP
- `RemoteAction`でPiPウィンドウにボタン追加
- Composeで`isInPictureInPictureMode`をリアクティブ監視

---

8種類のAndroidアプリテンプレート（動画機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Media3/ExoPlayer](https://zenn.dev/myougatheaxo/articles/android-compose-media3-exoplayer-2026)
- [動画録画](https://zenn.dev/myougatheaxo/articles/android-compose-video-recorder-2026)
- [フォアグラウンドサービス](https://zenn.dev/myougatheaxo/articles/android-compose-foreground-service-2026)
