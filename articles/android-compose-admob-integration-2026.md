---
title: "Google AdMob完全ガイド — Compose広告実装"
emoji: "💰"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "admob"]
published: true
---

## この記事で学べること

**Google AdMob**（バナー広告、インタースティシャル広告、リワード広告、Compose統合）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.android.gms:play-services-ads:23.6.0")
}
```

```xml
<!-- AndroidManifest.xml -->
<meta-data
    android:name="com.google.android.gms.ads.APPLICATION_ID"
    android:value="ca-app-pub-XXXXXXXX~YYYYYYYY" />
```

```kotlin
// Application
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        MobileAds.initialize(this)
    }
}
```

---

## バナー広告

```kotlin
@Composable
fun BannerAd(
    adUnitId: String = "ca-app-pub-3940256099942544/6300978111", // テスト用
    modifier: Modifier = Modifier
) {
    AndroidView(
        factory = { context ->
            AdView(context).apply {
                setAdSize(AdSize.BANNER)
                this.adUnitId = adUnitId
                loadAd(AdRequest.Builder().build())
            }
        },
        modifier = modifier.fillMaxWidth()
    )
}

// 画面下部に配置
@Composable
fun ScreenWithBanner() {
    Scaffold(
        bottomBar = {
            BannerAd(modifier = Modifier.fillMaxWidth())
        }
    ) { padding ->
        MainContent(Modifier.padding(padding))
    }
}
```

---

## インタースティシャル広告

```kotlin
class InterstitialAdManager(private val context: Context) {
    private var interstitialAd: InterstitialAd? = null

    fun load() {
        val adRequest = AdRequest.Builder().build()
        InterstitialAd.load(
            context,
            "ca-app-pub-3940256099942544/1033173712", // テスト用
            adRequest,
            object : InterstitialAdLoadCallback() {
                override fun onAdLoaded(ad: InterstitialAd) {
                    interstitialAd = ad
                }
                override fun onAdFailedToLoad(error: LoadAdError) {
                    interstitialAd = null
                    Log.e("AdMob", "Failed to load: ${error.message}")
                }
            }
        )
    }

    fun show(activity: Activity, onDismissed: () -> Unit) {
        interstitialAd?.let { ad ->
            ad.fullScreenContentCallback = object : FullScreenContentCallback() {
                override fun onAdDismissedFullScreenContent() {
                    interstitialAd = null
                    load() // 次の広告をプリロード
                    onDismissed()
                }
            }
            ad.show(activity)
        } ?: onDismissed() // 広告がない場合はスキップ
    }
}

// ViewModel統合
@HiltViewModel
class GameViewModel @Inject constructor() : ViewModel() {
    private var levelCount = 0

    fun onLevelComplete(showAd: () -> Unit, continueGame: () -> Unit) {
        levelCount++
        if (levelCount % 3 == 0) { // 3レベルごとに広告
            showAd()
        } else {
            continueGame()
        }
    }
}
```

---

## リワード広告

```kotlin
class RewardedAdManager(private val context: Context) {
    private var rewardedAd: RewardedAd? = null

    fun load() {
        val adRequest = AdRequest.Builder().build()
        RewardedAd.load(
            context,
            "ca-app-pub-3940256099942544/5224354917", // テスト用
            adRequest,
            object : RewardedAdLoadCallback() {
                override fun onAdLoaded(ad: RewardedAd) {
                    rewardedAd = ad
                }
                override fun onAdFailedToLoad(error: LoadAdError) {
                    rewardedAd = null
                }
            }
        )
    }

    fun show(activity: Activity, onRewarded: (RewardItem) -> Unit) {
        rewardedAd?.let { ad ->
            ad.fullScreenContentCallback = object : FullScreenContentCallback() {
                override fun onAdDismissedFullScreenContent() {
                    rewardedAd = null
                    load()
                }
            }
            ad.show(activity) { reward ->
                onRewarded(reward)
            }
        }
    }

    val isReady: Boolean get() = rewardedAd != null
}

// Compose画面
@Composable
fun RewardButton(
    adManager: RewardedAdManager,
    onRewarded: () -> Unit
) {
    val activity = LocalContext.current as Activity

    Button(
        onClick = {
            adManager.show(activity) { reward ->
                Log.d("AdMob", "Reward: ${reward.amount} ${reward.type}")
                onRewarded()
            }
        },
        enabled = adManager.isReady
    ) {
        Icon(Icons.Default.PlayArrow, contentDescription = null)
        Spacer(Modifier.width(8.dp))
        Text("動画を見てコインを獲得")
    }
}
```

---

## 広告なしプラン（課金連携）

```kotlin
@Composable
fun AdAwareScreen(
    isPremium: Boolean,
    content: @Composable (PaddingValues) -> Unit
) {
    Scaffold(
        bottomBar = {
            if (!isPremium) {
                BannerAd()
            }
        }
    ) { padding ->
        content(padding)
    }
}
```

---

## まとめ

| 広告タイプ | クラス | 用途 |
|-----------|--------|------|
| バナー | `AdView` | 常時表示 |
| インタースティシャル | `InterstitialAd` | 画面遷移時 |
| リワード | `RewardedAd` | 報酬付き動画 |

- `AndroidView`でバナー広告をComposeに統合
- テスト用広告IDで開発中はテスト広告表示
- インタースティシャルは適切な頻度で表示
- リワード広告でユーザー体験を損なわない収益化

---

8種類のAndroidアプリテンプレート（広告実装対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose⇔View相互運用](https://zenn.dev/myougatheaxo/articles/android-compose-compose-interop-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [状態管理MVI](https://zenn.dev/myougatheaxo/articles/android-compose-state-management-advanced-2026)
