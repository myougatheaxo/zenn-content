---
title: "TopAppBar完全ガイド — Composeでアプリバーを実装"
emoji: "🔝"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "material3"]
published: true
---

## この記事で学べること

Material3の**TopAppBar**の種類と実装方法、スクロール連動の設定まで解説します。

---

## CenterAlignedTopAppBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CenterAppBar() {
    Scaffold(
        topBar = {
            CenterAlignedTopAppBar(
                title = { Text("マイアプリ") },
                navigationIcon = {
                    IconButton(onClick = { /* メニュー */ }) {
                        Icon(Icons.Default.Menu, "メニュー")
                    }
                },
                actions = {
                    IconButton(onClick = { /* 検索 */ }) {
                        Icon(Icons.Default.Search, "検索")
                    }
                    IconButton(onClick = { /* その他 */ }) {
                        Icon(Icons.Default.MoreVert, "その他")
                    }
                }
            )
        }
    ) { padding ->
        // コンテンツ
    }
}
```

---

## TopAppBar（左揃え）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun StandardAppBar() {
    TopAppBar(
        title = { Text("設定") },
        navigationIcon = {
            IconButton(onClick = { /* 戻る */ }) {
                Icon(Icons.AutoMirrored.Filled.ArrowBack, "戻る")
            }
        }
    )
}
```

---

## MediumTopAppBar（中サイズ）

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MediumAppBar() {
    val scrollBehavior = TopAppBarDefaults.enterAlwaysScrollBehavior()

    Scaffold(
        topBar = {
            MediumTopAppBar(
                title = { Text("プロフィール") },
                scrollBehavior = scrollBehavior
            )
        },
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection)
    ) { padding ->
        LazyColumn(contentPadding = padding) {
            items(50) { Text("アイテム $it", Modifier.padding(16.dp)) }
        }
    }
}
```

スクロール時にタイトルが縮小。

---

## LargeTopAppBar

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun LargeAppBar() {
    val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

    Scaffold(
        topBar = {
            LargeTopAppBar(
                title = { Text("ニュースフィード") },
                scrollBehavior = scrollBehavior,
                colors = TopAppBarDefaults.largeTopAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    scrolledContainerColor = MaterialTheme.colorScheme.surface
                )
            )
        },
        modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection)
    ) { padding ->
        LazyColumn(contentPadding = padding) {
            items(100) { Text("ニュース $it", Modifier.padding(16.dp)) }
        }
    }
}
```

---

## ScrollBehaviorの種類

```kotlin
// スクロールで完全に隠れる
TopAppBarDefaults.enterAlwaysScrollBehavior()

// スクロールで縮小（完全には隠れない）
TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

// 固定（スクロールに反応しない）
TopAppBarDefaults.pinnedScrollBehavior()
```

---

## カスタムカラー

```kotlin
TopAppBar(
    title = { Text("カスタム") },
    colors = TopAppBarDefaults.topAppBarColors(
        containerColor = MaterialTheme.colorScheme.primaryContainer,
        titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer,
        navigationIconContentColor = MaterialTheme.colorScheme.onPrimaryContainer,
        actionIconContentColor = MaterialTheme.colorScheme.onPrimaryContainer
    )
)
```

---

## まとめ

- 4種類: `TopAppBar` / `CenterAlignedTopAppBar` / `MediumTopAppBar` / `LargeTopAppBar`
- `scrollBehavior`でスクロール連動
- `nestedScroll`でScaffoldと接続
- `navigationIcon`で戻るボタン/メニュー
- `actions`でアクションアイコン

---

8種類のAndroidアプリテンプレート（AppBar設計済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [BottomNavigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-bottom-nav-2026)
- [NavigationDrawer完全ガイド](https://zenn.dev/myougatheaxo/articles/android-compose-navigation-drawer-2026)
- [Material3コンポーネントカタログ](https://zenn.dev/myougatheaxo/articles/compose-material3-components-2026)
