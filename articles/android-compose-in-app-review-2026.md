---
title: "In-App Review API + Compose実装ガイド"
emoji: "⭐"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "googleplay"]
published: true
---

## この記事で学べること

Google Play **In-App Review API**をComposeアプリに統合する方法を解説します。

---

## セットアップ

```kotlin
dependencies {
    implementation("com.google.android.play:review-ktx:2.0.2")
}
```

---

## ReviewManagerの利用

```kotlin
class ReviewHelper(private val activity: Activity) {
    private val reviewManager = ReviewManagerFactory.create(activity)

    fun requestReview(
        onComplete: () -> Unit = {},
        onError: (Exception) -> Unit = {}
    ) {
        reviewManager.requestReviewFlow()
            .addOnSuccessListener { reviewInfo ->
                reviewManager.launchReviewFlow(activity, reviewInfo)
                    .addOnCompleteListener {
                        onComplete()
                    }
            }
            .addOnFailureListener { e ->
                onError(e)
            }
    }
}
```

---

## Composeでの利用

```kotlin
@Composable
fun ReviewPromptScreen(viewModel: MainViewModel = hiltViewModel()) {
    val context = LocalContext.current
    val activity = context as Activity
    val reviewHelper = remember { ReviewHelper(activity) }

    val launchCount by viewModel.launchCount.collectAsStateWithLifecycle()
    var reviewRequested by remember { mutableStateOf(false) }

    // 5回目の起動でレビュー促進
    LaunchedEffect(launchCount) {
        if (launchCount >= 5 && !reviewRequested) {
            reviewRequested = true
            reviewHelper.requestReview()
        }
    }
}

// ViewModelでの起動回数管理
@HiltViewModel
class MainViewModel @Inject constructor(
    private val dataStore: DataStore<Preferences>
) : ViewModel() {

    private val LAUNCH_COUNT = intPreferencesKey("launch_count")

    val launchCount = dataStore.data
        .map { it[LAUNCH_COUNT] ?: 0 }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0)

    init {
        viewModelScope.launch {
            dataStore.edit { prefs ->
                prefs[LAUNCH_COUNT] = (prefs[LAUNCH_COUNT] ?: 0) + 1
            }
        }
    }
}
```

---

## レビュー促進のベストプラクティス

```kotlin
@Composable
fun SmartReviewTrigger(
    hasCompletedTask: Boolean,
    daysSinceInstall: Int,
    reviewHelper: ReviewHelper
) {
    // ✅ 良いタイミング: タスク完了直後 + インストールから3日以上
    LaunchedEffect(hasCompletedTask) {
        if (hasCompletedTask && daysSinceInstall >= 3) {
            reviewHelper.requestReview()
        }
    }

    // ❌ 悪い例: アプリ起動直後に表示
    // ❌ 悪い例: エラー発生後に表示
    // ❌ 悪い例: 短期間に何度も表示
}
```

---

## まとめ

- `ReviewManagerFactory.create()`でReviewManager取得
- `requestReviewFlow()`→`launchReviewFlow()`の2ステップ
- レビューダイアログの表示はGoogleが制御（必ず表示される保証なし）
- 良いタイミング: タスク完了後、一定期間利用後
- 悪いタイミング: 初回起動、エラー後
- レビュー結果（送信/キャンセル）は取得不可

---

8種類のAndroidアプリテンプレート（レビュー促進機能対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開ガイド](https://zenn.dev/myougatheaxo/articles/android-google-play-release-2026)
- [DataStore設定管理](https://zenn.dev/myougatheaxo/articles/android-datastore-2026)
- [アプリ収益化戦略](https://zenn.dev/myougatheaxo/articles/android-app-monetization-2026)
