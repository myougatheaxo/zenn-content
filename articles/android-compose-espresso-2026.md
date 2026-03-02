---
title: "Espresso完全ガイド — UIテスト/ViewMatcher/RecyclerView/Idling Resource"
emoji: "☕"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Espresso**（UIテスト基本、ViewMatcher、RecyclerViewテスト、Idling Resource）を解説します。

---

## 基本テスト

```kotlin
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {

    @get:Rule
    val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Test
    fun loginFlow_withValidCredentials_navigatesToHome() {
        // テキスト入力
        onView(withId(R.id.email_input))
            .perform(typeText("test@example.com"), closeSoftKeyboard())

        onView(withId(R.id.password_input))
            .perform(typeText("password123"), closeSoftKeyboard())

        // ボタンクリック
        onView(withId(R.id.login_button))
            .perform(click())

        // 遷移確認
        onView(withText("ホーム"))
            .check(matches(isDisplayed()))
    }

    @Test
    fun loginButton_withEmptyEmail_isDisabled() {
        onView(withId(R.id.login_button))
            .check(matches(not(isEnabled())))
    }
}
```

---

## RecyclerViewテスト

```kotlin
@Test
fun recyclerView_scrollsToItem_andClicks() {
    // 特定位置にスクロール
    onView(withId(R.id.item_list))
        .perform(RecyclerViewActions.scrollToPosition<RecyclerView.ViewHolder>(15))

    // テキストを含むアイテムをクリック
    onView(withId(R.id.item_list))
        .perform(RecyclerViewActions.actionOnItem<RecyclerView.ViewHolder>(
            hasDescendant(withText("アイテム15")),
            click()
        ))

    // 詳細画面の確認
    onView(withId(R.id.detail_title))
        .check(matches(withText("アイテム15")))
}
```

---

## Idling Resource

```kotlin
class ApiIdlingResource : IdlingResource {
    private var callback: IdlingResource.ResourceCallback? = null

    @Volatile
    var isIdle = true
        set(value) {
            field = value
            if (value) callback?.onTransitionToIdle()
        }

    override fun getName() = "ApiIdlingResource"
    override fun isIdleNow() = isIdle
    override fun registerIdleTransitionCallback(callback: IdlingResource.ResourceCallback) {
        this.callback = callback
    }
}

// テストで使用
@RunWith(AndroidJUnit4::class)
class ApiTest {
    private val idlingResource = ApiIdlingResource()

    @Before
    fun setup() {
        IdlingRegistry.getInstance().register(idlingResource)
    }

    @After
    fun tearDown() {
        IdlingRegistry.getInstance().unregister(idlingResource)
    }

    @Test
    fun loadData_displaysResults() {
        onView(withId(R.id.load_button)).perform(click())
        // Idling Resourceが自動で待機
        onView(withId(R.id.result_list)).check(matches(isDisplayed()))
    }
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `onView` | View検索 |
| `perform` | 操作実行 |
| `check` | アサーション |
| `IdlingResource` | 非同期待機 |

- `ViewMatcher`でIDやテキストでView検索
- `ViewAction`でクリック・入力をシミュレーション
- `ViewAssertion`で表示状態を検証
- `IdlingResource`でAPI通信の完了を待機

---

8種類のAndroidアプリテンプレート（テスト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ui-test-2026)
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [ViewModel Testing](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-testing-2026)
