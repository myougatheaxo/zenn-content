---
title: "Material3 Adaptive完全ガイド — ListDetailPaneScaffold/NavigationSuiteScaffold"
emoji: "🖥️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

**Material3 Adaptive**（ListDetailPaneScaffold、NavigationSuiteScaffold、SupportingPaneScaffold、適応型UI）を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("androidx.compose.material3.adaptive:adaptive:1.1.0")
    implementation("androidx.compose.material3.adaptive:adaptive-layout:1.1.0")
    implementation("androidx.compose.material3.adaptive:adaptive-navigation:1.1.0")
    implementation("androidx.compose.material3:material3-adaptive-navigation-suite:1.3.1")
}
```

---

## ListDetailPaneScaffold

```kotlin
@OptIn(ExperimentalMaterial3AdaptiveApi::class)
@Composable
fun ListDetailScreen(items: List<Item>) {
    val navigator = rememberListDetailPaneScaffoldNavigator<Item>()

    ListDetailPaneScaffold(
        directive = navigator.scaffoldDirective,
        value = navigator.scaffoldValue,
        listPane = {
            AnimatedPane {
                LazyColumn {
                    items(items, key = { it.id }) { item ->
                        ListItem(
                            headlineContent = { Text(item.title) },
                            supportingContent = { Text(item.description) },
                            modifier = Modifier.clickable {
                                navigator.navigateTo(ListDetailPaneScaffoldRole.Detail, item)
                            }
                        )
                    }
                }
            }
        },
        detailPane = {
            AnimatedPane {
                navigator.currentDestination?.contentKey?.let { item ->
                    Column(Modifier.fillMaxSize().padding(16.dp)) {
                        Text(item.title, style = MaterialTheme.typography.headlineMedium)
                        Spacer(Modifier.height(8.dp))
                        Text(item.description)
                    }
                } ?: Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("アイテムを選択してください")
                }
            }
        }
    )
}
```

---

## NavigationSuiteScaffold

```kotlin
@Composable
fun AdaptiveNavApp() {
    var selectedItem by remember { mutableIntStateOf(0) }
    val navItems = listOf(
        NavItem("ホーム", Icons.Default.Home),
        NavItem("検索", Icons.Default.Search),
        NavItem("設定", Icons.Default.Settings)
    )

    NavigationSuiteScaffold(
        navigationSuiteItems = {
            navItems.forEachIndexed { index, item ->
                item(
                    icon = { Icon(item.icon, contentDescription = item.label) },
                    label = { Text(item.label) },
                    selected = selectedItem == index,
                    onClick = { selectedItem = index }
                )
            }
        }
    ) {
        when (selectedItem) {
            0 -> HomeContent()
            1 -> SearchContent()
            2 -> SettingsContent()
        }
    }
}

// NavigationSuiteScaffoldは画面サイズに応じて自動切替:
// Compact → NavigationBar (bottom)
// Medium → NavigationRail (side)
// Expanded → NavigationDrawer (permanent)
```

---

## SupportingPaneScaffold

```kotlin
@OptIn(ExperimentalMaterial3AdaptiveApi::class)
@Composable
fun SupportingPaneScreen() {
    val navigator = rememberSupportingPaneScaffoldNavigator()

    SupportingPaneScaffold(
        directive = navigator.scaffoldDirective,
        value = navigator.scaffoldValue,
        mainPane = {
            AnimatedPane {
                Column(Modifier.fillMaxSize().padding(16.dp)) {
                    Text("メインコンテンツ", style = MaterialTheme.typography.headlineMedium)
                    Button(onClick = {
                        navigator.navigateTo(SupportingPaneScaffoldRole.Supporting)
                    }) {
                        Text("サポートパネルを開く")
                    }
                }
            }
        },
        supportingPane = {
            AnimatedPane {
                Column(Modifier.fillMaxSize().padding(16.dp)) {
                    Text("サポート情報", style = MaterialTheme.typography.titleMedium)
                    Text("補足的な情報をここに表示")
                }
            }
        }
    )
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `ListDetailPaneScaffold` | リスト-詳細レイアウト |
| `NavigationSuiteScaffold` | 適応型ナビゲーション |
| `SupportingPaneScaffold` | メイン+サポートパネル |
| `AnimatedPane` | ペイン遷移アニメーション |

- Material3 Adaptiveで画面サイズに自動対応
- `ListDetailPaneScaffold`でタブレットUI自動構築
- `NavigationSuiteScaffold`でBottom/Rail/Drawer自動切替
- `AnimatedPane`でペイン遷移をスムーズに

---

8種類のAndroidアプリテンプレート（Adaptive UI対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [レスポンシブレイアウト](https://zenn.dev/myougatheaxo/articles/android-compose-responsive-layout-2026)
- [Material3コンポーネント](https://zenn.dev/myougatheaxo/articles/android-compose-material3-components-2026)
- [Navigation](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-nested-2026)
