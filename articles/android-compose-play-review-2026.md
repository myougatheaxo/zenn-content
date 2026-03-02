---
title: "Play Store ReviewAPI完全ガイド — アプリ内レビュー/レビュー誘導/タイミング制御"
emoji: "⭐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "playstore"]
published: true
---

## この記事で学べること

**Play Store ReviewAPI**（アプリ内レビュー表示、レビュー誘導タイミング、ユーザー体験を損なわない実装）を解説します。

---

## ReviewManager

```kotlin
class ReviewHelper @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val reviewManager = ReviewManagerFactory.create(context)

    fun requestReview(activity: Activity, onComplete: () -> Unit) {
        val request = reviewManager.requestReviewFlow()
        request.addOnCompleteListener { task ->
            if (task.isSuccessful) {
                val reviewInfo = task.result
                val flow = reviewManager.launchReviewFlow(activity, reviewInfo)
                flow.addOnCompleteListener { onComplete() }
            } else {
                onComplete()
            }
        }
    }
}
```

---

## レビュー誘導タイミング

```kotlin
class ReviewTrigger @Inject constructor(
    private val dataStore: DataStore<Preferences>,
    private val reviewHelper: ReviewHelper
) {
    companion object {
        private val LAUNCH_COUNT = intPreferencesKey("launch_count")
        private val LAST_REVIEW = longPreferencesKey("last_review_time")
        private val HAS_REVIEWED = booleanPreferencesKey("has_reviewed")
        private const val MIN_LAUNCHES = 5
        private const val MIN_DAYS_BETWEEN = 90L
    }

    suspend fun shouldShowReview(): Boolean {
        val prefs = dataStore.data.first()
        val launchCount = prefs[LAUNCH_COUNT] ?: 0
        val lastReview = prefs[LAST_REVIEW] ?: 0L
        val hasReviewed = prefs[HAS_REVIEWED] ?: false

        if (hasReviewed) return false
        if (launchCount < MIN_LAUNCHES) return false

        val daysSinceLastReview = TimeUnit.MILLISECONDS.toDays(
            System.currentTimeMillis() - lastReview
        )
        return daysSinceLastReview >= MIN_DAYS_BETWEEN
    }

    suspend fun incrementLaunchCount() {
        dataStore.edit { prefs ->
            prefs[LAUNCH_COUNT] = (prefs[LAUNCH_COUNT] ?: 0) + 1
        }
    }

    suspend fun markReviewed() {
        dataStore.edit { prefs ->
            prefs[LAST_REVIEW] = System.currentTimeMillis()
            prefs[HAS_REVIEWED] = true
        }
    }
}
```

---

## Compose UI

```kotlin
@Composable
fun ReviewPromptEffect(viewModel: ReviewViewModel = hiltViewModel()) {
    val shouldShow by viewModel.shouldShowReview.collectAsStateWithLifecycle(false)
    val activity = LocalContext.current as Activity

    LaunchedEffect(shouldShow) {
        if (shouldShow) {
            viewModel.requestReview(activity)
        }
    }
}

// 自然な誘導UI
@Composable
fun TaskCompletionDialog(onDismiss: () -> Unit, onReview: () -> Unit) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("タスク完了！🎉") },
        text = { Text("アプリを気に入っていただけましたら、レビューをお願いします。") },
        confirmButton = {
            TextButton(onClick = onReview) { Text("レビューする") }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) { Text("後で") }
        }
    )
}
```

---

## まとめ

| ポイント | 推奨 |
|---------|------|
| 最小起動回数 | 5回以上 |
| 間隔 | 90日以上 |
| タイミング | ポジティブ体験後 |
| 頻度 | 控えめに |

- `ReviewManagerFactory`でアプリ内レビューUI表示
- 起動回数・間隔でレビュー誘導タイミング制御
- ポジティブ体験（タスク完了等）の後に誘導
- レビュー済みフラグで再表示を防止

---

8種類のAndroidアプリテンプレート（レビュー機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [アプリ内レビュー](https://zenn.dev/myougatheaxo/articles/android-compose-in-app-review-2026)
- [アプリ内更新](https://zenn.dev/myougatheaxo/articles/android-compose-in-app-update-2026)
- [DataStore](https://zenn.dev/myougatheaxo/articles/android-compose-datastore-preferences-2026)
