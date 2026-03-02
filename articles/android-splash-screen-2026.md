---
title: "Android 12+ SplashScreen API完全ガイド — 起動画面の正しい実装"
emoji: "🚀"
type: "tech"
topics: ["android", "kotlin", "splashscreen", "ui"]
published: true
---

## この記事で学べること

Android 12からSplash Screen APIが標準化されました。古い自作スプラッシュ画面を**正しいAPI**に移行する方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.core:core-splashscreen:1.0.1")
}
```

---

## 基本実装（3ステップ）

### 1. テーマを設定

```xml
<!-- res/values/themes.xml -->
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/white</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_launcher_foreground</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>
```

### 2. Manifestに適用

```xml
<activity
    android:name=".MainActivity"
    android:theme="@style/Theme.App.Starting">
```

### 3. ActivityでinstallSplashScreen

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()
        super.onCreate(savedInstanceState)

        setContent {
            MyAppTheme {
                MainScreen()
            }
        }
    }
}
```

**`installSplashScreen()`は`super.onCreate()`の前に呼ぶ**のが重要。

---

## 初期化完了まで表示を延長

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()
        super.onCreate(savedInstanceState)

        val viewModel: MainViewModel by viewModels()

        splashScreen.setKeepOnScreenCondition {
            viewModel.isLoading.value
        }

        setContent {
            MyAppTheme {
                MainScreen(viewModel)
            }
        }
    }
}
```

```kotlin
class MainViewModel : ViewModel() {
    val isLoading = MutableStateFlow(true)

    init {
        viewModelScope.launch {
            // 初期データ読み込み
            loadInitialData()
            isLoading.value = false
        }
    }
}
```

`setKeepOnScreenCondition`が`true`の間、スプラッシュが表示され続けます。

---

## 終了アニメーション

```kotlin
splashScreen.setOnExitAnimationListener { splashScreenView ->
    val fadeOut = ObjectAnimator.ofFloat(
        splashScreenView.view,
        View.ALPHA,
        1f, 0f
    ).apply {
        duration = 500L
        interpolator = AccelerateInterpolator()
        doOnEnd { splashScreenView.remove() }
    }
    fadeOut.start()
}
```

---

## アニメーションアイコン

```xml
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenAnimatedIcon">@drawable/splash_animation</item>
    <item name="windowSplashScreenAnimationDuration">1000</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>
```

`AnimatedVectorDrawable`を使えば、アイコンがアニメーションします。

---

## ダークモード対応

```xml
<!-- res/values-night/themes.xml -->
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/dark_background</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_launcher_foreground</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>
```

`values-night`にテーマを定義すれば、ダークモード時に自動で切り替わります。

---

## まとめ

- `core-splashscreen`ライブラリを追加
- `installSplashScreen()`を`super.onCreate()`の前に呼ぶ
- `setKeepOnScreenCondition`で初期化完了まで延長
- `setOnExitAnimationListener`で終了アニメーション
- ダークモードは`values-night`で対応

---

8種類のAndroidアプリテンプレート（SplashScreen API対応済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [ダークモード完全対応ガイド](https://zenn.dev/myougatheaxo/articles/android-dark-mode-guide-2026)
- [Material3テーマ完全ガイド](https://zenn.dev/myougatheaxo/articles/material3-theme-guide-2026)
- [アプリアイコン設計ガイド](https://zenn.dev/myougatheaxo/articles/android-app-icon-guide-2026)
