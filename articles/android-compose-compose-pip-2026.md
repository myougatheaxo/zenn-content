---
title: "Compose PiP完全ガイド — ピクチャーインピクチャー/動画PiP/カスタムアクション"
emoji: "🖼️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "pip"]
published: true
---

## この記事で学べること

**Compose PiP**（ピクチャーインピクチャー、動画PiP、カスタムアクション、自動PiP遷移）を解説します。

---

## 基本PiP

```kotlin
class VideoActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { VideoScreen() }
    }

    override fun onUserLeaveHint() {
        super.onUserLeaveHint()
        enterPipMode()
    }

    private fun enterPipMode() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val params = PictureInPictureParams.Builder()
                .setAspectRatio(Rational(16, 9))
                .build()
            enterPictureInPictureMode(params)
        }
    }
}

// AndroidManifest.xml
// <activity android:supportsPictureInPicture="true"
//           android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation" />
```

---

## Compose PiP検出

```kotlin
@Composable
fun VideoScreen() {
    val context = LocalContext.current
    val activity = context as Activity
    val isInPipMode = rememberIsInPipMode()

    Column(Modifier.fillMaxSize()) {
        // 動画プレイヤー（PiPでも表示）
        VideoPlayer(Modifier.fillMaxWidth().aspectRatio(16f / 9f))

        // PiPモードでは非表示
        if (!isInPipMode) {
            Column(Modifier.padding(16.dp)) {
                Text("動画タイトル", style = MaterialTheme.typography.titleLarge)
                Text("説明文...")
                Button(onClick = { /* シェア */ }) { Text("共有") }
            }
        }
    }
}

@Composable
fun rememberIsInPipMode(): Boolean {
    val activity = LocalContext.current as Activity
    var isInPipMode by remember { mutableStateOf(activity.isInPictureInPictureMode) }

    DisposableEffect(activity) {
        val callback = object : ComponentCallbacks2 {
            override fun onConfigurationChanged(newConfig: Configuration) {
                isInPipMode = activity.isInPictureInPictureMode
            }
            override fun onTrimMemory(level: Int) {}
            override fun onLowMemory() {}
        }
        activity.registerComponentCallbacks(callback)
        onDispose { activity.unregisterComponentCallbacks(callback) }
    }

    return isInPipMode
}
```

---

## カスタムアクション

```kotlin
fun createPipParams(context: Context, isPlaying: Boolean): PictureInPictureParams {
    val actions = listOf(
        RemoteAction(
            Icon.createWithResource(context,
                if (isPlaying) R.drawable.ic_pause else R.drawable.ic_play),
            if (isPlaying) "一時停止" else "再生",
            if (isPlaying) "一時停止" else "再生",
            PendingIntent.getBroadcast(context, 0,
                Intent("TOGGLE_PLAY"), PendingIntent.FLAG_IMMUTABLE)
        )
    )

    return PictureInPictureParams.Builder()
        .setAspectRatio(Rational(16, 9))
        .setActions(actions)
        .build()
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `enterPictureInPictureMode` | PiP開始 |
| `PictureInPictureParams` | PiP設定 |
| `setAspectRatio` | アスペクト比 |
| `setActions` | カスタムアクション |

- `supportsPictureInPicture="true"`をManifestに追加
- `onUserLeaveHint`で自動PiP遷移
- PiPモード時は最小限のUIのみ表示
- `RemoteAction`でPiPウィンドウ内ボタンを追加

---

8種類のAndroidアプリテンプレート（メディア再生対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose VideoPlayer](https://zenn.dev/myougatheaxo/articles/android-compose-compose-video-player-2026)
- [Media3 Player](https://zenn.dev/myougatheaxo/articles/android-compose-media3-player-2026)
- [Compose MultiWindow](https://zenn.dev/myougatheaxo/articles/android-compose-compose-multi-window-2026)
