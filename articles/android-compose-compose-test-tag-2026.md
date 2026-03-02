---
title: "TestTag完全ガイド — テスト用タグ/testTag/Semantics Matcher"
emoji: "🏷️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**TestTag**（testTag、Semantics Matcher、UIテストでの要素特定）を解説します。

---

## testTag基本

```kotlin
// プロダクションコード
object TestTags {
    const val EMAIL_INPUT = "email_input"
    const val PASSWORD_INPUT = "password_input"
    const val LOGIN_BUTTON = "login_button"
    const val ERROR_TEXT = "error_text"
}

@Composable
fun LoginScreen(onLogin: (String, String) -> Unit) {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    Column(Modifier.padding(16.dp)) {
        OutlinedTextField(
            value = email, onValueChange = { email = it },
            label = { Text("メール") },
            modifier = Modifier.fillMaxWidth().testTag(TestTags.EMAIL_INPUT)
        )
        Spacer(Modifier.height(8.dp))
        OutlinedTextField(
            value = password, onValueChange = { password = it },
            label = { Text("パスワード") },
            visualTransformation = PasswordVisualTransformation(),
            modifier = Modifier.fillMaxWidth().testTag(TestTags.PASSWORD_INPUT)
        )
        Spacer(Modifier.height(16.dp))
        Button(
            onClick = { onLogin(email, password) },
            modifier = Modifier.fillMaxWidth().testTag(TestTags.LOGIN_BUTTON)
        ) { Text("ログイン") }
    }
}
```

---

## テストコード

```kotlin
@RunWith(AndroidJUnit4::class)
class LoginScreenTest {
    @get:Rule
    val composeRule = createComposeRule()

    @Test
    fun loginForm_displaysCorrectly() {
        composeRule.setContent { LoginScreen(onLogin = { _, _ -> }) }

        composeRule.onNodeWithTag(TestTags.EMAIL_INPUT).assertIsDisplayed()
        composeRule.onNodeWithTag(TestTags.PASSWORD_INPUT).assertIsDisplayed()
        composeRule.onNodeWithTag(TestTags.LOGIN_BUTTON).assertIsDisplayed()
    }

    @Test
    fun loginButton_clickWithInput() {
        var loginEmail = ""
        var loginPassword = ""

        composeRule.setContent {
            LoginScreen(onLogin = { e, p -> loginEmail = e; loginPassword = p })
        }

        composeRule.onNodeWithTag(TestTags.EMAIL_INPUT).performTextInput("test@example.com")
        composeRule.onNodeWithTag(TestTags.PASSWORD_INPUT).performTextInput("password123")
        composeRule.onNodeWithTag(TestTags.LOGIN_BUTTON).performClick()

        assert(loginEmail == "test@example.com")
        assert(loginPassword == "password123")
    }
}
```

---

## LazyColumnのtestTag

```kotlin
@Composable
fun ItemList(items: List<String>) {
    LazyColumn(Modifier.testTag("item_list")) {
        itemsIndexed(items) { index, item ->
            ListItem(
                headlineContent = { Text(item) },
                modifier = Modifier.testTag("item_$index")
            )
        }
    }
}

// テスト
@Test
fun itemList_scrollAndFind() {
    composeRule.setContent { ItemList((1..50).map { "Item $it" }) }

    composeRule.onNodeWithTag("item_list")
        .performScrollToIndex(30)
    composeRule.onNodeWithTag("item_30")
        .assertIsDisplayed()
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `testTag` | テスト用識別子 |
| `onNodeWithTag` | タグで要素検索 |
| `performTextInput` | テキスト入力 |
| `performClick` | クリック操作 |

- `testTag`でテスト時の要素特定を確実に
- オブジェクトでタグ定数を一元管理
- `onNodeWithTag`で要素にアクセス
- LazyColumnでもインデックス付きタグで特定可能

---

8種類のAndroidアプリテンプレート（テスト付き）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [UIテスト](https://zenn.dev/myougatheaxo/articles/android-compose-compose-ui-test-2026)
- [Semantics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-semantics-2026)
- [Hilt Testing](https://zenn.dev/myougatheaxo/articles/android-compose-hilt-testing-2026)
