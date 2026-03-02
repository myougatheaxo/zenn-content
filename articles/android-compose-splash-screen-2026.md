---
title: "SplashScreen完全ガイド — Core Splashscreen API/アニメーション/データ読込"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**SplashScreen**（Core Splashscreen API、アニメーション制御、データプリロード、テーマ設定）を解説します。

---

## 基本設定

```xml
<!-- res/values/themes.xml -->
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/splash_background</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash_icon</item>
    <item name="windowSplashScreenAnimationDuration">1000</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>
```

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity"
    android:theme="@style/Theme.App.Starting"
    android:exported="true">
```

---

## Activity実装

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()

        // データ読み込み完了までスプラッシュ表示を維持
        splashScreen.setKeepOnScreenCondition {
            viewModel.isLoading.value
        }

        // 終了アニメーション
        splashScreen.setOnExitAnimationListener { splashScreenView ->
            val fadeOut = ObjectAnimator.ofFloat(
                splashScreenView.view, View.ALPHA, 1f, 0f
            ).apply {
                duration = 500L
                interpolator = AccelerateInterpolator()
                doOnEnd { splashScreenView.remove() }
            }

            val scaleX = ObjectAnimator.ofFloat(
                splashScreenView.iconView, View.SCALE_X, 1f, 2f
            ).apply { duration = 500L }

            val scaleY = ObjectAnimator.ofFloat(
                splashScreenView.iconView, View.SCALE_Y, 1f, 2f
            ).apply { duration = 500L }

            AnimatorSet().apply {
                playTogether(fadeOut, scaleX, scaleY)
                start()
            }
        }

        super.onCreate(savedInstanceState)
        setContent { AppContent() }
    }
}
```

---

## ViewModel（データプリロード）

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val settingsRepository: SettingsRepository
) : ViewModel() {
    val isLoading = MutableStateFlow(true)

    init {
        viewModelScope.launch {
            try {
                // 並行でデータをプリロード
                coroutineScope {
                    launch { userRepository.preloadCurrentUser() }
                    launch { settingsRepository.preloadSettings() }
                }
            } finally {
                isLoading.value = false
            }
        }
    }
}
```

---

## Adaptive Icon対応

```xml
<!-- res/drawable/ic_splash_icon.xml -->
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:aapt="http://schemas.android.com/aapt">
    <aapt:attr name="android:drawable">
        <vector
            android:width="108dp"
            android:height="108dp"
            android:viewportWidth="108"
            android:viewportHeight="108">
            <!-- アイコンパス -->
        </vector>
    </aapt:attr>
    <!-- アニメーション定義 -->
</animated-vector>
```

---

## まとめ

| 機能 | 実装 |
|------|------|
| アイコン | `windowSplashScreenAnimatedIcon` |
| 背景色 | `windowSplashScreenBackground` |
| 表示維持 | `setKeepOnScreenCondition` |
| 終了アニメ | `setOnExitAnimationListener` |

- `installSplashScreen()`でCore Splashscreen APIを有効化
- `setKeepOnScreenCondition`でデータ読み込み完了まで表示維持
- カスタム終了アニメーションでスムーズな遷移
- Android 12+の仕様に準拠

---

8種類のAndroidアプリテンプレート（SplashScreen対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-type-safe-navigation-2026)
- [アニメーション](https://zenn.dev/myougatheaxo/articles/android-compose-animation-transition-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
