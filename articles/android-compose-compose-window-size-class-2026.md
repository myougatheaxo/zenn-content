---
title: "Compose WindowSizeClass完全ガイド — 画面サイズ分類/レスポンシブ/タブレット対応"
emoji: "📱"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "responsive"]
published: true
---

## この記事で学べること

**Compose WindowSizeClass**（画面サイズ分類、レスポンシブレイアウト、タブレット/折りたたみ対応）を解説します。

---

## 基本WindowSizeClass

```kotlin
// build.gradle
// implementation "androidx.compose.material3:material3-window-size-class:1.2.0"

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            val windowSizeClass = calculateWindowSizeClass(this)
            MyApp(windowSizeClass)
        }
    }
}

@Composable
fun MyApp(windowSizeClass: WindowSizeClass) {
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> PhoneLayout()
        WindowWidthSizeClass.Medium -> TabletLayout()
        WindowWidthSizeClass.Expanded -> DesktopLayout()
    }
}
```

---

## レスポンシブナビゲーション

```kotlin
@Composable
fun AdaptiveNavigation(windowSizeClass: WindowSizeClass, content: @Composable () -> Unit) {
    val useNavRail = windowSizeClass.widthSizeClass != WindowWidthSizeClass.Compact

    if (useNavRail) {
        Row {
            NavigationRail {
                NavigationRailItem(
                    selected = true,
                    onClick = {},
                    icon = { Icon(Icons.Default.Home, null) },
                    label = { Text("ホーム") }
                )
                NavigationRailItem(
                    selected = false,
                    onClick = {},
                    icon = { Icon(Icons.Default.Settings, null) },
                    label = { Text("設定") }
                )
            }
            content()
        }
    } else {
        Scaffold(
            bottomBar = {
                NavigationBar {
                    NavigationBarItem(selected = true, onClick = {},
                        icon = { Icon(Icons.Default.Home, null) }, label = { Text("ホーム") })
                    NavigationBarItem(selected = false, onClick = {},
                        icon = { Icon(Icons.Default.Settings, null) }, label = { Text("設定") })
                }
            }
        ) { padding ->
            Box(Modifier.padding(padding)) { content() }
        }
    }
}
```

---

## リスト-詳細レイアウト

```kotlin
@Composable
fun ListDetailLayout(windowSizeClass: WindowSizeClass) {
    var selectedId by remember { mutableStateOf<Long?>(null) }

    if (windowSizeClass.widthSizeClass == WindowWidthSizeClass.Expanded) {
        Row(Modifier.fillMaxSize()) {
            ItemList(
                onSelect = { selectedId = it },
                modifier = Modifier.weight(0.4f)
            )
            VerticalDivider()
            if (selectedId != null) {
                ItemDetail(selectedId!!, Modifier.weight(0.6f))
            } else {
                Box(Modifier.weight(0.6f), contentAlignment = Alignment.Center) {
                    Text("アイテムを選択してください")
                }
            }
        }
    } else {
        if (selectedId != null) {
            ItemDetail(selectedId!!, Modifier.fillMaxSize())
        } else {
            ItemList(onSelect = { selectedId = it }, Modifier.fillMaxSize())
        }
    }
}
```

---

## まとめ

| サイズクラス | 幅 | 対応デバイス |
|-------------|-----|-------------|
| `Compact` | < 600dp | スマートフォン |
| `Medium` | 600-840dp | タブレット/折りたたみ |
| `Expanded` | > 840dp | 大画面タブレット/PC |

- `calculateWindowSizeClass`で画面サイズを分類
- ナビゲーション: Compact→BottomBar、Medium/Expanded→NavigationRail
- リスト-詳細: Expanded→並列表示、Compact→画面遷移
- 折りたたみデバイスにも自動対応

---

8種類のAndroidアプリテンプレート（レスポンシブ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose AdaptiveLayout](https://zenn.dev/myougatheaxo/articles/android-compose-compose-adaptive-layout-2026)
- [Compose TwoPane](https://zenn.dev/myougatheaxo/articles/android-compose-compose-two-pane-2026)
- [Compose NavigationRail](https://zenn.dev/myougatheaxo/articles/android-compose-compose-navigation-rail-2026)
