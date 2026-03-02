---
title: "Compose LiveWallpaper完全ガイド — 動的壁紙/Canvas描画/センサー連動"
emoji: "🎆"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "wallpaper"]
published: true
---

## この記事で学べること

**Compose LiveWallpaper**（動的壁紙、WallpaperService、Canvas描画、センサー連動アニメーション）を解説します。

---

## WallpaperService

```kotlin
class ParticleWallpaperService : WallpaperService() {
    override fun onCreateEngine(): Engine = ParticleEngine()

    inner class ParticleEngine : Engine() {
        private val particles = mutableListOf<Particle>()
        private val paint = Paint().apply { color = Color.WHITE; alpha = 150 }
        private var running = true

        data class Particle(var x: Float, var y: Float, var speed: Float, var size: Float)

        override fun onSurfaceCreated(holder: SurfaceHolder) {
            super.onSurfaceCreated(holder)
            repeat(100) {
                particles.add(Particle(
                    x = Random.nextFloat() * holder.surfaceFrame.width(),
                    y = Random.nextFloat() * holder.surfaceFrame.height(),
                    speed = Random.nextFloat() * 3 + 1,
                    size = Random.nextFloat() * 5 + 2
                ))
            }
            startDrawing()
        }

        override fun onVisibilityChanged(visible: Boolean) {
            running = visible
            if (visible) startDrawing()
        }

        private fun startDrawing() {
            Thread {
                while (running) {
                    val canvas = surfaceHolder.lockCanvas() ?: continue
                    canvas.drawColor(android.graphics.Color.BLACK)
                    particles.forEach { p ->
                        canvas.drawCircle(p.x, p.y, p.size, paint)
                        p.y += p.speed
                        if (p.y > canvas.height) {
                            p.y = 0f
                            p.x = Random.nextFloat() * canvas.width
                        }
                    }
                    surfaceHolder.unlockCanvasAndPost(canvas)
                    Thread.sleep(16) // ~60fps
                }
            }.start()
        }

        override fun onSurfaceDestroyed(holder: SurfaceHolder) {
            running = false
            super.onSurfaceDestroyed(holder)
        }
    }
}
```

---

## Manifest + XML設定

```xml
<!-- AndroidManifest.xml -->
<service android:name=".ParticleWallpaperService"
    android:label="パーティクル壁紙"
    android:permission="android.permission.BIND_WALLPAPER"
    android:exported="true">
    <intent-filter>
        <action android:name="android.service.wallpaper.WallpaperService" />
    </intent-filter>
    <meta-data android:name="android.service.wallpaper"
               android:resource="@xml/wallpaper" />
</service>
```

```xml
<!-- res/xml/wallpaper.xml -->
<wallpaper xmlns:android="http://schemas.android.com/apk/res/android"
    android:thumbnail="@drawable/ic_wallpaper_preview"
    android:description="@string/wallpaper_description"
    android:settingsActivity=".WallpaperSettingsActivity" />
```

---

## 設定画面（Compose）

```kotlin
class WallpaperSettingsActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                WallpaperSettings()
            }
        }
    }
}

@Composable
fun WallpaperSettings() {
    var particleCount by remember { mutableFloatStateOf(100f) }
    var speed by remember { mutableFloatStateOf(2f) }

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Text("壁紙設定", style = MaterialTheme.typography.headlineMedium)

        Text("パーティクル数: ${particleCount.toInt()}")
        Slider(value = particleCount, onValueChange = { particleCount = it },
            valueRange = 10f..500f)

        Text("速度: ${speed.toInt()}")
        Slider(value = speed, onValueChange = { speed = it },
            valueRange = 1f..10f)
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `WallpaperService` | 動的壁紙サービス |
| `SurfaceHolder` | 描画サーフェス |
| `lockCanvas` | Canvas描画 |
| `settingsActivity` | 壁紙設定画面 |

- `WallpaperService`で動的壁紙を実装
- `lockCanvas()`/`unlockCanvasAndPost()`で描画ループ
- `onVisibilityChanged`で壁紙非表示時に描画を停止
- 設定画面はComposeで自由にUI構築可能

---

8種類のAndroidアプリテンプレート（Material3対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Canvas](https://zenn.dev/myougatheaxo/articles/android-compose-compose-canvas-2026)
- [Compose Animation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-animation-2026)
- [Compose ForegroundService](https://zenn.dev/myougatheaxo/articles/android-compose-compose-foreground-service-2026)
