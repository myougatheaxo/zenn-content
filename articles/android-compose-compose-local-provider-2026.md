---
title: "CompositionLocalProvider完全ガイド — ローカル値提供/テーマカスタム/依存注入"
emoji: "🎯"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "compositionlocal"]
published: true
---

## この記事で学べること

**CompositionLocalProvider**（CompositionLocal、provides、カスタムローカル値、テーマカスタマイズ）を解説します。

---

## CompositionLocal基本

```kotlin
val LocalAppConfig = staticCompositionLocalOf<AppConfig> {
    error("AppConfig not provided")
}

data class AppConfig(
    val apiBaseUrl: String,
    val isDebug: Boolean,
    val appVersion: String
)

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

@Composable
fun DeepNestedComponent() {
    val config = LocalAppConfig.current
    Text("API: ${config.apiBaseUrl}")
    if (config.isDebug) Text("Debug Mode", color = Color.Red)
}
```

---

## カスタムテーマ

```kotlin
val LocalSpacing = staticCompositionLocalOf { Spacing() }

data class Spacing(
    val small: Dp = 4.dp,
    val medium: Dp = 8.dp,
    val large: Dp = 16.dp,
    val extraLarge: Dp = 24.dp
)

@Composable
fun AppTheme(content: @Composable () -> Unit) {
    CompositionLocalProvider(
        LocalSpacing provides Spacing(),
        LocalContentColor provides Color.DarkGray
    ) {
        MaterialTheme { content() }
    }
}

@Composable
fun SpacedContent() {
    val spacing = LocalSpacing.current
    Column(Modifier.padding(spacing.large)) {
        Text("タイトル")
        Spacer(Modifier.height(spacing.medium))
        Text("コンテンツ")
    }
}
```

---

## 動的な値

```kotlin
val LocalUserSession = compositionLocalOf<UserSession?> { null }

@Composable
fun SessionAwareScreen() {
    val session = LocalUserSession.current

    if (session != null) {
        Text("ログイン中: ${session.userName}")
    } else {
        Text("未ログイン")
        Button(onClick = {}) { Text("ログイン") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `staticCompositionLocalOf` | 変更頻度低い値 |
| `compositionLocalOf` | 頻繁に変わる値 |
| `provides` | 値の提供 |
| `.current` | 値の取得 |

- `staticCompositionLocalOf`で変更頻度の低い設定値
- `compositionLocalOf`で頻繁に変わる動的値
- ツリー全体への暗黙的な値伝搬
- テーマ拡張（Spacing, Typography等）に最適

---

8種類のAndroidアプリテンプレート（テーマ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CompositionLocal](https://zenn.dev/myougatheaxo/articles/android-compose-compose-composition-local-2026)
- [State Hoisting](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-hoisting-2026)
- [Effect Handler](https://zenn.dev/myougatheaxo/articles/android-compose-compose-effect-handler-2026)
