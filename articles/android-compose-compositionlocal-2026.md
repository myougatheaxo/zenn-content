---
title: "CompositionLocal完全ガイド — テーマ/DI/暗黙的パラメータ受け渡し"
emoji: "🔗"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "compositionlocal"]
published: true
---

## この記事で学べること

**CompositionLocal**（staticCompositionLocalOf、compositionLocalOf、カスタムProvider、テーマ連携、テスト）を解説します。

---

## CompositionLocalとは

Composable間でデータを暗黙的に受け渡す仕組み。明示的な引数バケツリレーを避けられます。

```kotlin
// 組み込みのCompositionLocal例
@Composable
fun Example() {
    val context = LocalContext.current          // Context
    val density = LocalDensity.current          // Density
    val config = LocalConfiguration.current     // Configuration
    val lifecycle = LocalLifecycleOwner.current // LifecycleOwner
}
```

---

## カスタムCompositionLocal

```kotlin
// staticCompositionLocalOf: 値が変わることが稀な場合（再コンポジション範囲が広い）
val LocalAppConfig = staticCompositionLocalOf<AppConfig> {
    error("No AppConfig provided")
}

// compositionLocalOf: 値が頻繁に変わる場合（変更箇所のみ再コンポジション）
val LocalUserSession = compositionLocalOf<UserSession?> { null }

data class AppConfig(
    val apiBaseUrl: String,
    val isDebug: Boolean,
    val appVersion: String
)

data class UserSession(
    val userId: String,
    val displayName: String,
    val isAdmin: Boolean
)
```

---

## Provider

```kotlin
@Composable
fun App() {
    val appConfig = AppConfig(
        apiBaseUrl = BuildConfig.API_URL,
        isDebug = BuildConfig.DEBUG,
        appVersion = BuildConfig.VERSION_NAME
    )
    val userSession = remember { mutableStateOf<UserSession?>(null) }

    CompositionLocalProvider(
        LocalAppConfig provides appConfig,
        LocalUserSession provides userSession.value
    ) {
        AppNavigation()
    }
}

// 子Composableでアクセス
@Composable
fun SettingsScreen() {
    val config = LocalAppConfig.current
    val session = LocalUserSession.current

    Column(Modifier.padding(16.dp)) {
        Text("Version: ${config.appVersion}")
        session?.let { Text("User: ${it.displayName}") }
    }
}
```

---

## カスタムテーマ値

```kotlin
@Immutable
data class AppSpacing(
    val small: Dp = 4.dp,
    val medium: Dp = 8.dp,
    val large: Dp = 16.dp,
    val extraLarge: Dp = 24.dp
)

val LocalSpacing = staticCompositionLocalOf { AppSpacing() }

// MaterialThemeの拡張として使用
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    CompositionLocalProvider(LocalSpacing provides AppSpacing()) {
        MaterialTheme(content = content)
    }
}

// アクセサー拡張プロパティ
val MaterialTheme.spacing: AppSpacing
    @Composable @ReadOnlyComposable
    get() = LocalSpacing.current

// 使用
@Composable
fun MyScreen() {
    Column(Modifier.padding(MaterialTheme.spacing.large)) {
        Text("Hello", Modifier.padding(bottom = MaterialTheme.spacing.medium))
    }
}
```

---

## テスト

```kotlin
@Test
fun testWithCustomCompositionLocal() {
    val testConfig = AppConfig(
        apiBaseUrl = "https://test.api.com",
        isDebug = true,
        appVersion = "1.0.0-test"
    )

    composeTestRule.setContent {
        CompositionLocalProvider(LocalAppConfig provides testConfig) {
            SettingsScreen()
        }
    }

    composeTestRule.onNodeWithText("Version: 1.0.0-test").assertIsDisplayed()
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `compositionLocalOf` | 頻繁に変わる値 |
| `staticCompositionLocalOf` | 変更が稀な値 |
| `CompositionLocalProvider` | 値の提供 |
| `.current` | 値の取得 |

- `staticCompositionLocalOf`は変更時にサブツリー全体を再コンポジション
- `compositionLocalOf`は変更時に読み取り箇所のみ再コンポジション
- テーマ拡張（Spacing、Elevation等）に最適
- テストでは`CompositionLocalProvider`で差し替え

---

8種類のAndroidアプリテンプレート（カスタムテーマ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theming-2026)
- [ダークモード](https://zenn.dev/myougatheaxo/articles/android-compose-theme-dark-mode-2026)
- [カスタムComposable](https://zenn.dev/myougatheaxo/articles/android-compose-custom-composable-2026)
