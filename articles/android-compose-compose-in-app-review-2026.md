---
title: "In-App Review完全ガイド — レビューリクエスト/タイミング制御/Compose統合"
emoji: "⭐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "playstore"]
published: true
---

## この記事で学べること

**In-App Review**（ReviewManager、レビューリクエスト、最適タイミング、Compose統合）を解説します。

---

## ReviewManager基本

```kotlin
@Composable
fun InAppReviewTrigger() {
    val context = LocalContext.current
    val activity = context as Activity
    val reviewManager = remember { ReviewManagerFactory.create(context) }

    LaunchedEffect(Unit) {
        val request = reviewManager.requestReviewFlow()
        request.addOnSuccessListener { reviewInfo ->
            reviewManager.launchReviewFlow(activity, reviewInfo)
        }
    }
}
```

---

## タイミング制御

```kotlin
class ReviewHelper @Inject constructor(
    @ApplicationContext private val context: Context,
    private val dataStore: DataStore<Preferences>
) {
    private val LAUNCH_COUNT = intPreferencesKey("launch_count")
    private val LAST_REVIEW = longPreferencesKey("last_review_time")
    private val REVIEW_THRESHOLD = 5
    private val REVIEW_INTERVAL_DAYS = 90L

    suspend fun shouldShowReview(): Boolean {
        val prefs = dataStore.data.first()
        val launchCount = prefs[LAUNCH_COUNT] ?: 0
        val lastReview = prefs[LAST_REVIEW] ?: 0L
        val daysSinceLastReview = (System.currentTimeMillis() - lastReview) / (1000 * 60 * 60 * 24)

        return launchCount >= REVIEW_THRESHOLD && daysSinceLastReview >= REVIEW_INTERVAL_DAYS
    }

    suspend fun incrementLaunchCount() {
        dataStore.edit { it[LAUNCH_COUNT] = (it[LAUNCH_COUNT] ?: 0) + 1 }
    }

    suspend fun markReviewShown() {
        dataStore.edit {
            it[LAST_REVIEW] = System.currentTimeMillis()
            it[LAUNCH_COUNT] = 0
        }
    }
}

@HiltViewModel
class MainViewModel @Inject constructor(
    private val reviewHelper: ReviewHelper
) : ViewModel() {
    val shouldShowReview = MutableStateFlow(false)

    init {
        viewModelScope.launch {
            reviewHelper.incrementLaunchCount()
            shouldShowReview.value = reviewHelper.shouldShowReview()
        }
    }

    fun onReviewShown() {
        viewModelScope.launch { reviewHelper.markReviewShown() }
    }
}
```

---

## Compose統合

```kotlin
@Composable
fun MainScreenWithReview(viewModel: MainViewModel = hiltViewModel()) {
    val shouldReview by viewModel.shouldShowReview.collectAsStateWithLifecycle()
    val context = LocalContext.current
    val activity = context as Activity

    if (shouldReview) {
        LaunchedEffect(Unit) {
            val manager = ReviewManagerFactory.create(context)
            manager.requestReviewFlow().addOnSuccessListener { info ->
                manager.launchReviewFlow(activity, info).addOnCompleteListener {
                    viewModel.onReviewShown()
                }
            }
        }
    }

    // メインコンテンツ
    Scaffold { innerPadding ->
        Box(Modifier.padding(innerPadding)) { Text("メイン画面") }
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `ReviewManagerFactory` | Manager生成 |
| `requestReviewFlow` | レビュー情報取得 |
| `launchReviewFlow` | レビューUI表示 |
| DataStore | タイミング管理 |

- Google Play In-App Reviewで自然なレビュー依頼
- 起動回数+経過日数でタイミング制御
- ユーザー体験を損なわない適切な頻度
- レビューの結果はアプリ側から取得不可

---

8種類のAndroidアプリテンプレート（Play Store対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [In-App Update](https://zenn.dev/myougatheaxo/articles/android-compose-compose-in-app-update-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-2026)
- [設定画面](https://zenn.dev/myougatheaxo/articles/android-compose-compose-settings-screen-2026)
