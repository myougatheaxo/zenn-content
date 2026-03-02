---
title: "Onboarding完全ガイド — ページャー/ステップUI/初回起動判定"
emoji: "👋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "onboarding"]
published: true
---

## この記事で学べること

**Onboarding**（HorizontalPager、ページインジケーター、DataStoreによる初回起動判定）を解説します。

---

## Onboardingページャー

```kotlin
data class OnboardingPage(val title: String, val description: String, val icon: ImageVector)

@OptIn(ExperimentalFoundationApi::class)
@Composable
fun OnboardingScreen(onComplete: () -> Unit) {
    val pages = listOf(
        OnboardingPage("ようこそ", "アプリの主な機能を紹介します", Icons.Default.Waving),
        OnboardingPage("簡単操作", "直感的なUIで誰でも使えます", Icons.Default.TouchApp),
        OnboardingPage("さあ始めよう", "今すぐ使い始めましょう", Icons.Default.Rocket)
    )

    val pagerState = rememberPagerState(pageCount = { pages.size })
    val scope = rememberCoroutineScope()

    Column(Modifier.fillMaxSize()) {
        HorizontalPager(state = pagerState, modifier = Modifier.weight(1f)) { page ->
            OnboardingPageContent(pages[page])
        }

        // インジケーター
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.Center) {
            repeat(pages.size) { index ->
                Box(
                    Modifier
                        .padding(4.dp)
                        .size(if (pagerState.currentPage == index) 12.dp else 8.dp)
                        .clip(CircleShape)
                        .background(
                            if (pagerState.currentPage == index)
                                MaterialTheme.colorScheme.primary
                            else Color.LightGray
                        )
                )
            }
        }

        // ボタン
        Row(Modifier.fillMaxWidth().padding(16.dp)) {
            if (pagerState.currentPage > 0) {
                TextButton(onClick = { scope.launch { pagerState.animateScrollToPage(pagerState.currentPage - 1) } }) {
                    Text("戻る")
                }
            }
            Spacer(Modifier.weight(1f))
            if (pagerState.currentPage < pages.size - 1) {
                Button(onClick = { scope.launch { pagerState.animateScrollToPage(pagerState.currentPage + 1) } }) {
                    Text("次へ")
                }
            } else {
                Button(onClick = onComplete) { Text("始める") }
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
        Text(page.title, style = MaterialTheme.typography.headlineMedium, fontWeight = FontWeight.Bold)
        Spacer(Modifier.height(16.dp))
        Text(page.description, style = MaterialTheme.typography.bodyLarge, textAlign = TextAlign.Center)
    }
}
```

---

## 初回起動判定

```kotlin
class OnboardingPreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val Context.dataStore by preferencesDataStore("onboarding")
    private val COMPLETED_KEY = booleanPreferencesKey("onboarding_completed")

    val isCompleted: Flow<Boolean> = context.dataStore.data.map { it[COMPLETED_KEY] ?: false }

    suspend fun setCompleted() {
        context.dataStore.edit { it[COMPLETED_KEY] = true }
    }
}

@Composable
fun AppEntryPoint(prefs: OnboardingPreferences = hiltViewModel<AppViewModel>().prefs) {
    val isCompleted by prefs.isCompleted.collectAsStateWithLifecycle(initialValue = null)

    when (isCompleted) {
        null -> { /* Loading */ }
        false -> OnboardingScreen(onComplete = {
            CoroutineScope(Dispatchers.IO).launch { prefs.setCompleted() }
        })
        true -> MainScreen()
    }
}
```

---

## まとめ

| コンポーネント | 用途 |
|--------------|------|
| `HorizontalPager` | ページスワイプ |
| `PagerState` | ページ状態管理 |
| `DataStore` | 初回起動フラグ |
| インジケーター | 現在ページ表示 |

- `HorizontalPager`でスワイプ可能なOnboarding
- ページインジケーターで進捗表示
- DataStoreで初回起動フラグを永続化
- 完了後はメイン画面へ遷移

---

8種類のAndroidアプリテンプレート（Onboarding対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Pager](https://zenn.dev/myougatheaxo/articles/android-compose-compose-pager-2026)
- [SplashScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-splash-screen-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
