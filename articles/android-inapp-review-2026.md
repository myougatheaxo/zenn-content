---
title: "アプリ内レビュー実装ガイド — In-App Review APIでストア評価を増やす"
emoji: "⭐"
type: "tech"
topics: ["android", "kotlin", "googleplay", "review"]
published: true
---

## この記事で学べること

Google Play In-App Review APIを使えば、**アプリを離れずにストアレビュー**を促せます。ストア評価を上げる正しい実装方法を解説します。

---

## セットアップ

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.android.play:review-ktx:2.0.1")
}
```

---

## 基本実装

```kotlin
class ReviewManager(private val activity: Activity) {
    private val manager = ReviewManagerFactory.create(activity)

    fun requestReview() {
        val request = manager.requestReviewFlow()
        request.addOnCompleteListener { task ->
            if (task.isSuccessful) {
                val reviewInfo = task.result
                val flow = manager.launchReviewFlow(activity, reviewInfo)
                flow.addOnCompleteListener {
                    // レビューフロー完了（結果は取得不可）
                }
            }
        }
    }
}
```

---

## Composeでの実装

```kotlin
@Composable
fun ReviewPrompt() {
    val context = LocalContext.current
    val activity = context as Activity
    val reviewManager = remember { ReviewManagerFactory.create(context) }

    LaunchedEffect(Unit) {
        val request = reviewManager.requestReviewFlow()
        request.addOnCompleteListener { task ->
            if (task.isSuccessful) {
                reviewManager.launchReviewFlow(activity, task.result)
            }
        }
    }
}
```

---

## 適切なタイミング

レビューは**ユーザーが満足しているタイミング**で促すのが重要です。

```kotlin
class ReviewTrigger(
    private val dataStore: DataStore<Preferences>
) {
    private val LAUNCH_COUNT_KEY = intPreferencesKey("launch_count")
    private val LAST_REVIEW_KEY = longPreferencesKey("last_review_time")

    suspend fun shouldShowReview(): Boolean {
        val prefs = dataStore.data.first()
        val launchCount = prefs[LAUNCH_COUNT_KEY] ?: 0
        val lastReview = prefs[LAST_REVIEW_KEY] ?: 0L
        val daysSinceLastReview = (System.currentTimeMillis() - lastReview) /
            (1000 * 60 * 60 * 24)

        return launchCount >= 5 && daysSinceLastReview >= 30
    }

    suspend fun incrementLaunchCount() {
        dataStore.edit { prefs ->
            val current = prefs[LAUNCH_COUNT_KEY] ?: 0
            prefs[LAUNCH_COUNT_KEY] = current + 1
        }
    }

    suspend fun markReviewShown() {
        dataStore.edit { prefs ->
            prefs[LAST_REVIEW_KEY] = System.currentTimeMillis()
        }
    }
}
```

### ベストプラクティス

| タイミング | 良い例 |
|-----------|--------|
| タスク完了後 | 習慣トラッカーで7日連続記録達成 |
| 成果表示後 | 家計簿で月間レポート確認後 |
| 一定期間後 | アプリ起動5回目以降 |

### やってはいけないこと

| NG | 理由 |
|----|------|
| 初回起動でレビュー | ユーザーがまだ価値を感じていない |
| 頻繁にレビュー表示 | Googleがレビューダイアログを表示しなくなる |
| レビュー結果で分岐 | APIは結果を返さない（仕様） |

---

## テスト

```kotlin
// デバッグビルドではFakeReviewManagerを使う
val manager = if (BuildConfig.DEBUG) {
    FakeReviewManager(context)
} else {
    ReviewManagerFactory.create(context)
}
```

`FakeReviewManager`はダイアログを模擬表示します。本番のレビューダイアログは**内部テストトラック以上**でのみ動作します。

---

## まとめ

- `review-ktx`ライブラリを追加
- `requestReviewFlow` → `launchReviewFlow`の2ステップ
- ユーザーが**満足しているタイミング**で表示
- 結果は取得不可（Google仕様）
- テストには`FakeReviewManager`を使う
- 30日以上の間隔を空ける

---

8種類のAndroidアプリテンプレート（レビュー促進機能追加可能）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Google Play公開チェックリスト](https://zenn.dev/myougatheaxo/articles/google-play-publish-guide-2026)
- [DataStore移行ガイド](https://zenn.dev/myougatheaxo/articles/android-datastore-migration-2026)
- [APKサイズ最適化](https://zenn.dev/myougatheaxo/articles/android-apk-optimization-2026)
