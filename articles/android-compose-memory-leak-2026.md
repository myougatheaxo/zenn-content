---
title: "メモリリーク対策完全ガイド — LeakCanary/よくあるパターン/修正方法"
emoji: "💧"
type: "tech"
topics: ["android", "kotlin", "performance", "debugging"]
published: true
---

## この記事で学べること

**メモリリーク対策**（LeakCanary、よくあるリークパターン、修正方法、Compose固有の注意点）を解説します。

---

## LeakCanary設定

```kotlin
// build.gradle.kts
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
}
// デバッグビルドで自動的にリーク検出開始（設定不要）
```

---

## よくあるリークパターン

```kotlin
// ❌ パターン1: Contextの長期保持
class DataManager(private val context: Context) {  // Activity Contextを保持 → リーク
    companion object {
        var instance: DataManager? = null
    }
}

// ✅ 修正: Application Contextを使用
class DataManager @Inject constructor(
    @ApplicationContext private val context: Context
) { /* ... */ }

// ❌ パターン2: コールバック未解除
class LocationTracker(context: Context) {
    private val locationManager = context.getSystemService(Context.LOCATION_SERVICE) as LocationManager

    fun startTracking(listener: LocationListener) {
        locationManager.requestLocationUpdates("gps", 0, 0f, listener)
        // stopTracking() を呼ばないとリーク
    }
}

// ✅ 修正: ライフサイクル連動
class LocationTracker @Inject constructor(
    @ApplicationContext private val context: Context
) : DefaultLifecycleObserver {
    private var listener: LocationListener? = null

    override fun onDestroy(owner: LifecycleOwner) {
        listener?.let { locationManager.removeUpdates(it) }
        listener = null
    }
}
```

---

## Compose固有の注意点

```kotlin
// ❌ Composable内でContextを長期保持
@Composable
fun BadExample() {
    val context = LocalContext.current
    val manager = remember { HeavyManager(context) }  // Activity Contextをキャプチャ
}

// ✅ Application Contextを使用
@Composable
fun GoodExample() {
    val context = LocalContext.current.applicationContext
    val manager = remember { HeavyManager(context) }
}

// ❌ LaunchedEffectで外部スコープをキャプチャ
@Composable
fun BadLaunchedEffect(activity: Activity) {
    LaunchedEffect(Unit) {
        delay(30_000)
        activity.doSomething()  // activityをキャプチャ → リーク
    }
}

// ❌ rememberで大きなオブジェクト
@Composable
fun BadRemember() {
    val bitmap = remember { loadLargeBitmap() }  // メモリ圧迫
}

// ✅ DisposableEffectでクリーンアップ
@Composable
fun CleanupExample() {
    val callback = remember { MyCallback() }
    DisposableEffect(Unit) {
        EventBus.register(callback)
        onDispose { EventBus.unregister(callback) }
    }
}
```

---

## ViewModelでの注意

```kotlin
// ❌ ViewModelでView/Activityを保持
class BadViewModel : ViewModel() {
    var activity: Activity? = null  // リーク確定
}

// ❌ ViewModelでContextをフィールドに保持
class BadViewModel2(private val context: Context) : ViewModel()

// ✅ SavedStateHandle or ApplicationContextを使用
class GoodViewModel @Inject constructor(
    @ApplicationContext private val appContext: Context,
    private val savedStateHandle: SavedStateHandle
) : ViewModel()
```

---

## まとめ

| リーク原因 | 修正方法 |
|-----------|----------|
| Activity Context保持 | Application Context使用 |
| コールバック未解除 | `DisposableEffect`/`onDestroy` |
| 匿名クラスの外部参照 | `WeakReference` or ライフサイクル連動 |
| ViewModel→View参照 | 絶対禁止 |

- LeakCanaryをdebug依存に追加するだけで自動検出
- `@ApplicationContext`でActivity Contextリークを防止
- `DisposableEffect`で登録/解除をペアで管理
- ViewModelにView/Activity参照を絶対に持たない

---

8種類のAndroidアプリテンプレート（メモリ最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [デバッグツール](https://zenn.dev/myougatheaxo/articles/android-compose-debug-tools-2026)
- [パフォーマンス](https://zenn.dev/myougatheaxo/articles/android-compose-stability-performance-2026)
- [Lifecycle](https://zenn.dev/myougatheaxo/articles/android-compose-lifecycle-aware-2026)
