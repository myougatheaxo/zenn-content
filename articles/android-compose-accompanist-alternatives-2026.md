---
title: "Accompanist移行ガイド — 公式API代替/Permissions/SystemUiController"
emoji: "🔄"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accompanist"]
published: true
---

## この記事で学べること

**Accompanist移行**（Permissions→公式API、SystemUiController→enableEdgeToEdge、Pager→HorizontalPager、Navigation Animation→公式API）を解説します。

---

## Accompanist非推奨の背景

Accompanist のほとんどの機能は Compose 公式に移行済み。2024年以降、新規プロジェクトではAccompanistを使わず公式APIを使いましょう。

---

## Permissions

```kotlin
// ❌ Accompanist (非推奨)
// val permissionState = rememberPermissionState(Manifest.permission.CAMERA)

// ✅ 公式 ActivityResultContracts
@Composable
fun CameraPermissionScreen() {
    var hasPermission by remember { mutableStateOf(false) }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted -> hasPermission = granted }

    LaunchedEffect(Unit) {
        launcher.launch(Manifest.permission.CAMERA)
    }

    if (hasPermission) {
        CameraContent()
    } else {
        Text("カメラ権限が必要です")
    }
}
```

---

## SystemUiController → enableEdgeToEdge

```kotlin
// ❌ Accompanist SystemUiController (非推奨)
// val systemUiController = rememberSystemUiController()
// systemUiController.setStatusBarColor(Color.Transparent)

// ✅ 公式 enableEdgeToEdge (Activity)
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge()  // ステータスバー/ナビバー透過
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Scaffold(
                    modifier = Modifier.fillMaxSize(),
                    contentWindowInsets = WindowInsets(0)
                ) { innerPadding ->
                    Content(Modifier.padding(innerPadding))
                }
            }
        }
    }
}
```

---

## Pager → HorizontalPager

```kotlin
// ❌ Accompanist HorizontalPager (非推奨)
// HorizontalPager(count = 3, state = pagerState) { page -> ... }

// ✅ 公式 HorizontalPager (foundation)
@Composable
fun TabPager() {
    val pagerState = rememberPagerState(pageCount = { 3 })

    Column {
        TabRow(selectedTabIndex = pagerState.currentPage) {
            listOf("タブ1", "タブ2", "タブ3").forEachIndexed { index, title ->
                Tab(
                    selected = pagerState.currentPage == index,
                    onClick = { /* coroutineScope.launch { pagerState.animateScrollToPage(index) } */ },
                    text = { Text(title) }
                )
            }
        }

        HorizontalPager(state = pagerState) { page ->
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                Text("ページ ${page + 1}")
            }
        }
    }
}
```

---

## Navigation Animation → 公式

```kotlin
// ❌ Accompanist AnimatedNavHost (非推奨)
// AnimatedNavHost(navController, startDestination = "home") { ... }

// ✅ 公式 NavHost (アニメーション内蔵)
NavHost(navController, startDestination = "home") {
    composable(
        route = "home",
        enterTransition = { slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Left) },
        exitTransition = { slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Left) }
    ) { HomeScreen() }
}
```

---

## まとめ

| Accompanist | 公式代替 |
|-------------|---------|
| Permissions | `ActivityResultContracts` |
| SystemUiController | `enableEdgeToEdge()` |
| HorizontalPager | `foundation.pager` |
| AnimatedNavHost | `navigation-compose` |
| SwipeRefresh | `material3.PullToRefreshBox` |
| FlowLayout | `foundation.FlowRow` |

- 新規プロジェクトはAccompanist不要
- 既存プロジェクトは段階的に公式APIへ移行
- `foundation`と`material3`に大半が統合済み

---

8種類のAndroidアプリテンプレート（最新API対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Edge-to-Edge](https://zenn.dev/myougatheaxo/articles/android-compose-edge-to-edge-2026)
- [パーミッション](https://zenn.dev/myougatheaxo/articles/android-compose-permission-handling-2026)
- [Pull-to-Refresh](https://zenn.dev/myougatheaxo/articles/android-compose-pull-to-refresh-2026)
