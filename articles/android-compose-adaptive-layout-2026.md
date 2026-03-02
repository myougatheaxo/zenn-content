---
title: "Compose Adaptive Layout — タブレット・折りたたみ対応レスポンシブUI"
emoji: "📐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "tablet"]
published: true
---

## この記事で学べること

タブレットや折りたたみデバイスに対応する**レスポンシブレイアウト**をComposeで実装する方法を解説します。

---

## 画面サイズの分類

```kotlin
enum class WindowSizeClass {
    COMPACT,   // スマホ縦（~600dp）
    MEDIUM,    // タブレット縦・折りたたみ（600-840dp）
    EXPANDED   // タブレット横（840dp~）
}

@Composable
fun calculateWindowSizeClass(): WindowSizeClass {
    val configuration = LocalConfiguration.current
    return when {
        configuration.screenWidthDp < 600 -> WindowSizeClass.COMPACT
        configuration.screenWidthDp < 840 -> WindowSizeClass.MEDIUM
        else -> WindowSizeClass.EXPANDED
    }
}
```

---

## レスポンシブレイアウト

```kotlin
@Composable
fun AdaptiveScreen(viewModel: TaskViewModel) {
    val windowSize = calculateWindowSizeClass()

    when (windowSize) {
        WindowSizeClass.COMPACT -> {
            // スマホ: 1カラム
            TaskListScreen(viewModel)
        }
        WindowSizeClass.MEDIUM -> {
            // タブレット縦: リスト + 詳細パネル
            Row {
                TaskListScreen(
                    viewModel,
                    modifier = Modifier.weight(0.4f)
                )
                TaskDetailPanel(
                    viewModel,
                    modifier = Modifier.weight(0.6f)
                )
            }
        }
        WindowSizeClass.EXPANDED -> {
            // タブレット横: ワイドレイアウト
            Row {
                NavigationRail { /* ... */ }
                TaskListScreen(
                    viewModel,
                    modifier = Modifier.weight(0.3f)
                )
                TaskDetailPanel(
                    viewModel,
                    modifier = Modifier.weight(0.7f)
                )
            }
        }
    }
}
```

---

## NavigationBar vs NavigationRail

```kotlin
@Composable
fun AdaptiveNavigation(
    windowSize: WindowSizeClass,
    content: @Composable () -> Unit
) {
    when (windowSize) {
        WindowSizeClass.COMPACT -> {
            Scaffold(
                bottomBar = {
                    NavigationBar {
                        // BottomNavigation（スマホ）
                    }
                }
            ) { padding ->
                Box(Modifier.padding(padding)) { content() }
            }
        }
        else -> {
            Row {
                NavigationRail {
                    // サイドナビ（タブレット）
                }
                content()
            }
        }
    }
}
```

| デバイス | ナビゲーション |
|---------|-------------|
| スマホ | `NavigationBar`（下部） |
| タブレット縦 | `NavigationRail`（左側） |
| タブレット横 | `NavigationDrawer`（左側展開） |

---

## リスト-詳細パターン

```kotlin
@Composable
fun ListDetailLayout(
    windowSize: WindowSizeClass,
    list: @Composable (Modifier) -> Unit,
    detail: @Composable (Modifier) -> Unit
) {
    if (windowSize == WindowSizeClass.COMPACT) {
        // スマホ: ナビゲーションで切り替え
        var showDetail by remember { mutableStateOf(false) }
        if (showDetail) {
            detail(Modifier.fillMaxSize())
        } else {
            list(Modifier.fillMaxSize())
        }
    } else {
        // タブレット: 横並び
        Row(Modifier.fillMaxSize()) {
            list(Modifier.weight(0.4f))
            VerticalDivider()
            detail(Modifier.weight(0.6f))
        }
    }
}
```

---

## Previewでの確認

```kotlin
@Preview(name = "Phone", device = Devices.PIXEL_7)
@Preview(name = "Tablet", device = Devices.PIXEL_TABLET)
@Preview(name = "Foldable", device = Devices.PIXEL_FOLD)
@Composable
fun AdaptiveScreenPreview() {
    MyAppTheme {
        AdaptiveScreen(viewModel = FakeViewModel())
    }
}
```

---

## まとめ

- `WindowSizeClass`で画面サイズを3段階に分類
- COMPACT: 1カラム + BottomNavigation
- MEDIUM/EXPANDED: マルチカラム + NavigationRail
- リスト-詳細パターンが最も汎用的
- Previewで複数デバイスを同時確認

---

8種類のAndroidアプリテンプレート（レスポンシブ対応可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Composeレイアウト入門](https://zenn.dev/myougatheaxo/articles/compose-layout-basics-2026)
- [Compose Preview完全ガイド](https://zenn.dev/myougatheaxo/articles/compose-preview-guide-2026)
- [Edge-to-Edge表示完全ガイド](https://zenn.dev/myougatheaxo/articles/android-edge-to-edge-2026)
