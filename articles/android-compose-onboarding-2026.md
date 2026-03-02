---
title: "オンボーディング画面の実装 — ComposeでHorizontalPager初回起動ガイド"
emoji: "👋"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "ui"]
published: true
---

## この記事で学べること

**HorizontalPager**を使ったオンボーディング（初回起動ガイド）画面の実装方法を解説します。

---

## オンボーディングデータ

```kotlin
data class OnboardingPage(
    val title: String,
    val description: String,
    val icon: ImageVector
)

val onboardingPages = listOf(
    OnboardingPage(
        "タスクを管理",
        "日々のタスクを簡単に追加・管理できます。",
        Icons.Default.CheckCircle
    ),
    OnboardingPage(
        "リマインダー",
        "大事なタスクを忘れないよう通知します。",
        Icons.Default.Notifications
    ),
    OnboardingPage(
        "統計で振り返り",
        "完了したタスクの統計を確認できます。",
        Icons.Default.BarChart
    )
)
```

---

## オンボーディング画面

```kotlin
@Composable
fun OnboardingScreen(onComplete: () -> Unit) {
    val pagerState = rememberPagerState(pageCount = { onboardingPages.size })
    val scope = rememberCoroutineScope()

    Column(
        Modifier.fillMaxSize().padding(16.dp)
    ) {
        // スキップボタン
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.End) {
            TextButton(onClick = onComplete) {
                Text("スキップ")
            }
        }

        // ページコンテンツ
        HorizontalPager(
            state = pagerState,
            modifier = Modifier.weight(1f)
        ) { page ->
            OnboardingPageContent(onboardingPages[page])
        }

        // インジケーター
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.Center
        ) {
            repeat(onboardingPages.size) { index ->
                val color = if (pagerState.currentPage == index)
                    MaterialTheme.colorScheme.primary
                else MaterialTheme.colorScheme.outline.copy(alpha = 0.3f)

                Box(
                    Modifier
                        .padding(4.dp)
                        .size(if (pagerState.currentPage == index) 12.dp else 8.dp)
                        .clip(CircleShape)
                        .background(color)
                )
            }
        }

        Spacer(Modifier.height(32.dp))

        // ボタン
        if (pagerState.currentPage == onboardingPages.size - 1) {
            Button(
                onClick = onComplete,
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("はじめる")
            }
        } else {
            Button(
                onClick = {
                    scope.launch {
                        pagerState.animateScrollToPage(pagerState.currentPage + 1)
                    }
                },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("次へ")
            }
        }

        Spacer(Modifier.height(16.dp))
    }
}

@Composable
fun OnboardingPageContent(page: OnboardingPage) {
    Column(
        Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            page.icon,
            null,
            modifier = Modifier.size(120.dp),
            tint = MaterialTheme.colorScheme.primary
        )
        Spacer(Modifier.height(32.dp))
        Text(
            page.title,
            style = MaterialTheme.typography.headlineMedium,
            textAlign = TextAlign.Center
        )
        Spacer(Modifier.height(16.dp))
        Text(
            page.description,
            style = MaterialTheme.typography.bodyLarge,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}
```

---

## 初回表示の制御（DataStore）

```kotlin
class OnboardingPrefs(private val dataStore: DataStore<Preferences>) {
    val isCompleted: Flow<Boolean> = dataStore.data.map { prefs ->
        prefs[booleanPreferencesKey("onboarding_completed")] ?: false
    }

    suspend fun setCompleted() {
        dataStore.edit { prefs ->
            prefs[booleanPreferencesKey("onboarding_completed")] = true
        }
    }
}

// Navigation
@Composable
fun AppNavigation(prefs: OnboardingPrefs) {
    val isCompleted by prefs.isCompleted.collectAsStateWithLifecycle(initialValue = null)

    when (isCompleted) {
        null -> { /* Loading */ }
        false -> OnboardingScreen(onComplete = {
            scope.launch { prefs.setCompleted() }
        })
        true -> MainScreen()
    }
}
```

---

## まとめ

- `HorizontalPager`でスワイプ可能なオンボーディング
- ページインジケーターで現在位置を表示
- 最終ページで「はじめる」ボタンに切り替え
- DataStoreで初回表示済みフラグを永続化
- スキップボタンでいつでも完了可能

---

8種類のAndroidアプリテンプレート（オンボーディング追加可能な設計）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [TabRow + HorizontalPager](https://zenn.dev/myougatheaxo/articles/android-compose-tab-pager-2026)
- [DataStore完全ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-guide-2026)
- [Navigation完全ガイド](https://zenn.dev/myougatheaxo/articles/android-navigation-compose-2026)
