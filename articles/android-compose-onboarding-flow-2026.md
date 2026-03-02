---
title: "オンボーディング完全ガイド — ページャー/スキップ/初回のみ表示"
emoji: "👋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**オンボーディング**（HorizontalPager、ドットインジケーター、初回のみ表示、DataStore連携）を解説します。

---

## オンボーディング画面

```kotlin
data class OnboardingPage(
    val title: String,
    val description: String,
    val icon: ImageVector
)

@Composable
fun OnboardingScreen(onComplete: () -> Unit) {
    val pages = listOf(
        OnboardingPage("ようこそ", "アプリの主な機能を紹介します", Icons.Default.Waving),
        OnboardingPage("簡単操作", "直感的なUIで誰でも使えます", Icons.Default.TouchApp),
        OnboardingPage("始めましょう", "さっそく使ってみましょう", Icons.Default.RocketLaunch)
    )

    val pagerState = rememberPagerState(pageCount = { pages.size })
    val scope = rememberCoroutineScope()
    val isLastPage = pagerState.currentPage == pages.lastIndex

    Column(Modifier.fillMaxSize()) {
        // スキップボタン
        Row(Modifier.fillMaxWidth().padding(16.dp), horizontalArrangement = Arrangement.End) {
            TextButton(onClick = onComplete) { Text("スキップ") }
        }

        // ページャー
        HorizontalPager(state = pagerState, modifier = Modifier.weight(1f)) { page ->
            OnboardingPageContent(pages[page])
        }

        // ドットインジケーター + ボタン
        Row(
            Modifier.fillMaxWidth().padding(24.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            // ドット
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                repeat(pages.size) { index ->
                    Box(
                        Modifier
                            .size(if (pagerState.currentPage == index) 10.dp else 8.dp)
                            .background(
                                if (pagerState.currentPage == index) MaterialTheme.colorScheme.primary
                                else MaterialTheme.colorScheme.outline,
                                CircleShape
                            )
                    )
                }
            }

            // ボタン
            Button(onClick = {
                if (isLastPage) onComplete()
                else scope.launch { pagerState.animateScrollToPage(pagerState.currentPage + 1) }
            }) {
                Text(if (isLastPage) "始める" else "次へ")
            }
        }
    }
}

@Composable
fun OnboardingPageContent(page: OnboardingPage) {
    Column(
        Modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(page.icon, null, Modifier.size(120.dp), tint = MaterialTheme.colorScheme.primary)
        Spacer(Modifier.height(32.dp))
        Text(page.title, style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))
        Text(page.description, style = MaterialTheme.typography.bodyLarge, textAlign = TextAlign.Center)
    }
}
```

---

## 初回のみ表示

```kotlin
class OnboardingRepository @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {
    val isOnboardingCompleted: Flow<Boolean> = dataStore.data
        .map { it[booleanPreferencesKey("onboarding_completed")] ?: false }

    suspend fun completeOnboarding() {
        dataStore.edit { it[booleanPreferencesKey("onboarding_completed")] = true }
    }
}

@Composable
fun AppEntryPoint(
    viewModel: MainViewModel = hiltViewModel()
) {
    val isOnboardingDone by viewModel.isOnboardingCompleted.collectAsStateWithLifecycle(null)

    when (isOnboardingDone) {
        null -> { /* ローディング */ }
        false -> OnboardingScreen(onComplete = { viewModel.completeOnboarding() })
        true -> MainScreen()
    }
}
```

---

## まとめ

| 要素 | 実装 |
|------|------|
| ページャー | `HorizontalPager` |
| インジケーター | ドットRow |
| 初回制御 | `DataStore<Preferences>` |
| ナビゲーション | スキップ/次へ/始める |

- `HorizontalPager`でスワイプ可能なオンボーディング
- DataStoreで初回表示フラグを永続化
- スキップボタンで即座に本画面へ
- ドットインジケーターで現在位置を表示

---

8種類のAndroidアプリテンプレート（オンボーディング実装済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [TabRow/Pager](https://zenn.dev/myougatheaxo/articles/android-compose-tabrow-viewpager-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
- [SplashScreen](https://zenn.dev/myougatheaxo/articles/android-compose-splash-screen-2026)
