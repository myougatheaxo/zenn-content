---
title: "Compose AccessibilityTesting完全ガイド — Semantics検証/TalkBack/コントラスト/自動テスト"
emoji: "♿"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "accessibility"]
published: true
---

## この記事で学べること

**Compose AccessibilityTesting**（Semanticsテスト、TalkBack検証、コントラストチェック、自動テスト）を解説します。

---

## Semanticsテスト

```kotlin
@Test
fun buttonHasCorrectSemantics() {
    composeTestRule.setContent {
        Button(onClick = {}) {
            Icon(Icons.Default.Add, contentDescription = null)
            Text("追加")
        }
    }

    composeTestRule.onNodeWithText("追加")
        .assertIsDisplayed()
        .assertHasClickAction()
}

@Test
fun imageHasContentDescription() {
    composeTestRule.setContent {
        Image(
            painter = painterResource(R.drawable.photo),
            contentDescription = "プロフィール写真"
        )
    }

    composeTestRule.onNodeWithContentDescription("プロフィール写真")
        .assertExists()
}

@Test
fun switchHasStateDescription() {
    composeTestRule.setContent {
        var checked by remember { mutableStateOf(false) }
        Switch(
            checked = checked,
            onCheckedChange = { checked = it },
            modifier = Modifier.semantics {
                stateDescription = if (checked) "オン" else "オフ"
            }
        )
    }

    composeTestRule.onNode(hasStateDescription("オフ")).assertExists()
}
```

---

## タッチターゲットテスト

```kotlin
@Test
fun touchTargetMinSize() {
    composeTestRule.setContent {
        IconButton(onClick = {}) {
            Icon(Icons.Default.Close, contentDescription = "閉じる")
        }
    }

    // 48dp以上のタッチターゲット
    composeTestRule.onNodeWithContentDescription("閉じる")
        .assertTouchHeightIsAtLeast(48.dp)
        .assertTouchWidthIsAtLeast(48.dp)
}

@Test
fun checkHeadingSemantics() {
    composeTestRule.setContent {
        Text("セクションタイトル",
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.semantics { heading() })
    }

    composeTestRule.onNode(isHeading()).assertExists()
}
```

---

## アクセシビリティチェック

```kotlin
// Espresso Accessibility Checks
@get:Rule
val composeTestRule = createComposeRule()

@Before
fun enableAccessibilityChecks() {
    AccessibilityChecks.enable()
        .setRunChecksFromRootView(true)
        .setThrowExceptionFor(AccessibilityCheckResult.AccessibilityCheckResultType.ERROR)
}

@Test
fun screenPassesAccessibilityChecks() {
    composeTestRule.setContent {
        MaterialTheme {
            Column(Modifier.padding(16.dp)) {
                Text("タイトル", style = MaterialTheme.typography.headlineMedium,
                    modifier = Modifier.semantics { heading() })
                Button(onClick = {}) { Text("アクション") }
                Image(painter = painterResource(R.drawable.icon),
                    contentDescription = "アイコン")
            }
        }
    }

    // Espressoのアクセシビリティチェックが自動実行
    composeTestRule.onNodeWithText("アクション").performClick()
}
```

---

## まとめ

| API | 用途 |
|-----|------|
| `assertHasClickAction` | クリック可能検証 |
| `onNodeWithContentDescription` | 説明テスト |
| `assertTouchHeightIsAtLeast` | タッチサイズ |
| `AccessibilityChecks` | 自動チェック |

- `Semantics`でアクセシビリティ情報をテスト
- タッチターゲット48dp以上を検証
- `AccessibilityChecks`で自動的に問題を検出
- `contentDescription`の設定漏れをテストで防止

---

8種類のAndroidアプリテンプレート（アクセシビリティ対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Compose Accessibility](https://zenn.dev/myougatheaxo/articles/android-compose-compose-accessibility-2026)
- [Compose Semantics](https://zenn.dev/myougatheaxo/articles/android-compose-compose-semantics-2026)
- [Compose TestingComposeRule](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-compose-rule-2026)
