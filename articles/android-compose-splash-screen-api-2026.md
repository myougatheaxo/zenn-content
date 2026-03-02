---
title: "SplashScreen API完全ガイド — Android 12+対応"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "splash"]
published: true
---

## この記事で学べること

**SplashScreen API**（Android 12+、アイコンアニメーション、初期データ読み込み待ち）を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.core:core-splashscreen:1.0.1")
}
```

---

## 基本設定

```xml
<!-- res/values/themes.xml -->
<style name="Theme.App.Splash" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/splash_bg</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash_icon</item>
    <item name="windowSplashScreenAnimationDuration">300</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>

<!-- AndroidManifest.xml -->
<!-- android:theme="@style/Theme.App.Splash" -->
```

---

## Activity実装

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {

    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        // installSplashScreen は setContentView の前に呼ぶ
        val splashScreen = installSplashScreen()

        // 初期データ読み込みが完了するまでスプラッシュ表示
        splashScreen.setKeepOnScreenCondition {
            viewModel.isLoading.value
        }

        // 終了アニメーション
        splashScreen.setOnExitAnimationListener { splashScreenView ->
            val fadeOut = ObjectAnimator.ofFloat(
                splashScreenView.view, View.ALPHA, 1f, 0f
            ).apply {
                duration = 300
                interpolator = AccelerateInterpolator()
                doOnEnd { splashScreenView.remove() }
            }
            fadeOut.start()
        }

        super.onCreate(savedInstanceState)

        setContent {
            AppTheme {
                AppNavigation()
            }
        }
    }
}
```

---

## ViewModel（初期ロード）

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val authRepository: AuthRepository,
    private val settingsRepository: SettingsRepository
) : ViewModel() {

    private val _isLoading = MutableStateFlow(true)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    private val _startDestination = MutableStateFlow<String?>(null)
    val startDestination: StateFlow<String?> = _startDestination.asStateFlow()

    init {
        viewModelScope.launch {
            try {
                // 並行で初期化
                val isLoggedIn = async { authRepository.isLoggedIn() }
                val hasCompletedOnboarding = async { settingsRepository.hasCompletedOnboarding() }

                _startDestination.value = when {
                    !hasCompletedOnboarding.await() -> "onboarding"
                    !isLoggedIn.await() -> "login"
                    else -> "home"
                }
            } finally {
                _isLoading.value = false
            }
        }
    }
}
```

---

## Navigation連携

```kotlin
@Composable
fun AppNavigation(viewModel: MainViewModel = hiltViewModel()) {
    val startDestination by viewModel.startDestination.collectAsStateWithLifecycle()

    startDestination?.let { destination ->
        val navController = rememberNavController()

        NavHost(navController = navController, startDestination = destination) {
            composable("onboarding") {
                OnboardingScreen(onComplete = {
                    navController.navigate("login") {
                        popUpTo("onboarding") { inclusive = true }
                    }
                })
            }
            composable("login") {
                LoginScreen(onLoginSuccess = {
                    navController.navigate("home") {
                        popUpTo("login") { inclusive = true }
                    }
                })
            }
            composable("home") {
                HomeScreen()
            }
        }
    }
}
```

---

## アニメーションアイコン

```xml
<!-- res/drawable/ic_splash_animated.xml -->
<animated-vector
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_splash_icon">
    <target
        android:name="icon_group"
        android:animation="@animator/splash_icon_anim" />
</animated-vector>

<!-- res/animator/splash_icon_anim.xml -->
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <objectAnimator
        android:propertyName="scaleX"
        android:valueFrom="0"
        android:valueTo="1"
        android:duration="500"
        android:interpolator="@android:interpolator/overshoot" />
    <objectAnimator
        android:propertyName="scaleY"
        android:valueFrom="0"
        android:valueTo="1"
        android:duration="500"
        android:interpolator="@android:interpolator/overshoot" />
</set>
```

---

## まとめ

- `installSplashScreen()`を`setContentView`/`setContent`の前に呼ぶ
- `setKeepOnScreenCondition`で初期データ読み込み待ち
- `setOnExitAnimationListener`で終了アニメーション
- Android 12+は自動スプラッシュ→APIで制御
- `animated-vector`でアイコンアニメーション
- ViewModelで認証/設定チェック→startDestination決定

---

8種類のAndroidアプリテンプレート（SplashScreen実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [App Startup](https://zenn.dev/myougatheaxo/articles/android-compose-app-startup-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
- [Firebase Auth](https://zenn.dev/myougatheaxo/articles/android-compose-firebase-auth-2026)
