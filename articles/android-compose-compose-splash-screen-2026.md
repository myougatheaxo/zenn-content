---
title: "SplashScreen完全ガイド — Core Splash Screen API/アニメーション/ブランディング"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "splashscreen"]
published: true
---

## この記事で学べること

**SplashScreen**（Core Splash Screen API、アニメーション付きスプラッシュ、初期化処理）を解説します。

---

## Core Splash Screen API

```kotlin
// themes.xml
// <style name="Theme.App.Starting" parent="Theme.SplashScreen">
//     <item name="windowSplashScreenBackground">@color/splash_bg</item>
//     <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash</item>
//     <item name="windowSplashScreenAnimationDuration">1000</item>
//     <item name="postSplashScreenTheme">@style/Theme.App</item>
// </style>

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()

        // 初期化完了まで表示を維持
        var isReady = false
        splashScreen.setKeepOnScreenCondition { !isReady }

        super.onCreate(savedInstanceState)

        // 初期化処理
        lifecycleScope.launch {
            delay(1000) // データ読み込み等
            isReady = true
        }

        setContent {
            AppTheme { MainScreen() }
        }
    }
}
```

---

## 終了アニメーション

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()

        splashScreen.setOnExitAnimationListener { splashScreenView ->
            // フェードアウト
            val fadeOut = ObjectAnimator.ofFloat(splashScreenView.view, View.ALPHA, 1f, 0f)
            fadeOut.duration = 500L
            fadeOut.interpolator = AccelerateInterpolator()
            fadeOut.doOnEnd { splashScreenView.remove() }

            // スケール
            val scaleX = ObjectAnimator.ofFloat(splashScreenView.iconView, View.SCALE_X, 1f, 0f)
            val scaleY = ObjectAnimator.ofFloat(splashScreenView.iconView, View.SCALE_Y, 1f, 0f)

            AnimatorSet().apply {
                playTogether(fadeOut, scaleX, scaleY)
                start()
            }
        }

        super.onCreate(savedInstanceState)
        setContent { AppTheme { MainScreen() } }
    }
}
```

---

## ViewModel連携

```kotlin
@HiltViewModel
class StartupViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val settingsRepository: SettingsRepository
) : ViewModel() {
    private val _isReady = MutableStateFlow(false)
    val isReady = _isReady.asStateFlow()

    init {
        viewModelScope.launch {
            // 並行初期化
            coroutineScope {
                launch { userRepository.loadCurrentUser() }
                launch { settingsRepository.loadSettings() }
            }
            _isReady.value = true
        }
    }
}

// MainActivity
class MainActivity : ComponentActivity() {
    private val viewModel: StartupViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()
        splashScreen.setKeepOnScreenCondition { !viewModel.isReady.value }
        super.onCreate(savedInstanceState)
        setContent { AppTheme { MainScreen() } }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `installSplashScreen()` | Splash Screen設定 |
| `setKeepOnScreenCondition` | 表示維持条件 |
| `setOnExitAnimationListener` | 終了アニメーション |
| `postSplashScreenTheme` | メインテーマ |

- Core Splash Screen APIでAndroid 12対応
- `setKeepOnScreenCondition`で初期化完了まで待機
- `setOnExitAnimationListener`でカスタム遷移
- ViewModelで初期化ロジックを管理

---

8種類のAndroidアプリテンプレート（SplashScreen対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Edge-to-Edge](https://zenn.dev/myougatheaxo/articles/android-compose-compose-edge-to-edge-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-type-safe-2026)
- [ViewModel](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-savedstate-2026)
