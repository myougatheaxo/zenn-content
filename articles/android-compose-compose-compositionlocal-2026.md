---
title: "CompositionLocal完全ガイド — staticOf/compositionLocalOf/テーマ/DI代替"
emoji: "🔌"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "compose"]
published: true
---

## この記事で学べること

**CompositionLocal**（staticCompositionLocalOf、compositionLocalOf、カスタムテーマ、コンテキスト共有）を解説します。

---

## 基本的な使い方

```kotlin
// 定義
val LocalAppConfig = staticCompositionLocalOf<AppConfig> {
    error("AppConfig not provided")
}

data class AppConfig(
    val apiBaseUrl: String,
    val isDebug: Boolean,
    val appVersion: String
)

// 提供
@Composable
fun AppRoot() {
    val config = AppConfig(
        apiBaseUrl = "https://api.example.com",
        isDebug = BuildConfig.DEBUG,
        appVersion = "1.0.0"
    )

    CompositionLocalProvider(LocalAppConfig provides config) {
        MainScreen()
    }
}

// 消費
@Composable
fun DebugBanner() {
    val config = LocalAppConfig.current
    if (config.isDebug) {
        Surface(color = Color.Red) {
            Text("DEBUG v${config.appVersion}", color = Color.White)
        }
    }
}
```

---

## compositionLocalOf vs staticCompositionLocalOf

```kotlin
// 頻繁に変わる値 → compositionLocalOf（変更時に消費者のみ再コンポーズ）
val LocalUserPreferences = compositionLocalOf { UserPreferences() }

// 変わらない値 → staticCompositionLocalOf（変更時にサブツリー全体を再コンポーズ）
val LocalAnalytics = staticCompositionLocalOf<AnalyticsTracker> {
    error("AnalyticsTracker not provided")
}

@Composable
fun AppProviders(content: @Composable () -> Unit) {
    val prefs by viewModel.preferences.collectAsStateWithLifecycle(UserPreferences())
    val analytics = remember { AnalyticsTracker() }

    CompositionLocalProvider(
        LocalUserPreferences provides prefs,
        LocalAnalytics provides analytics
    ) {
        content()
    }
}
```

---

## カスタムテーマシステム

```kotlin
data class AppSpacing(
    val small: Dp = 4.dp,
    val medium: Dp = 8.dp,
    val large: Dp = 16.dp,
    val extraLarge: Dp = 32.dp
)

val LocalSpacing = staticCompositionLocalOf { AppSpacing() }

// MaterialTheme拡張
object AppTheme {
    val spacing: AppSpacing
        @Composable get() = LocalSpacing.current
}

// 使用
@Composable
fun SpacedContent() {
    Column(Modifier.padding(AppTheme.spacing.large)) {
        Text("タイトル")
        Spacer(Modifier.height(AppTheme.spacing.medium))
        Text("本文")
    }
}
```

---

## まとめ

| 種類 | 変更時の動作 |
|------|------------|
| `compositionLocalOf` | 消費者のみ再コンポーズ |
| `staticCompositionLocalOf` | サブツリー全体再コンポーズ |

- `CompositionLocal`でコンポーズツリーにデータを暗黙的に渡す
- `compositionLocalOf`は頻繁に変わる値に使用
- `staticCompositionLocalOf`は不変データに使用
- テーマ拡張やDI的なパターンに活用

---

8種類のAndroidアプリテンプレート（テーマ設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [Hilt DI](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-dependency-injection-2026)
