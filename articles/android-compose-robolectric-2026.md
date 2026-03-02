---
title: "Robolectric完全ガイド — JVMテスト/Androidフレームワーク/高速テスト"
emoji: "🤖"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "testing"]
published: true
---

## この記事で学べること

**Robolectric**（JVM上のAndroidテスト、Context/Resource利用、高速テスト実行）を解説します。

---

## 基本設定

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.robolectric:robolectric:4.12")
    testImplementation("androidx.test:core:1.6.1")
    testImplementation("androidx.test.ext:junit:1.2.1")
}

android {
    testOptions {
        unitTests {
            isIncludeAndroidResources = true
        }
    }
}
```

---

## テスト例

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [34])
class NotificationHelperTest {

    private val context = ApplicationProvider.getApplicationContext<Context>()

    @Test
    fun createChannel_createsNotificationChannel() {
        val helper = NotificationHelper(context)
        helper.createChannels()

        val manager = context.getSystemService(NotificationManager::class.java)
        val channel = manager.getNotificationChannel("general")

        assertNotNull(channel)
        assertEquals("一般", channel.name)
    }

    @Test
    fun sharedPreferences_savesAndLoads() {
        val prefs = context.getSharedPreferences("test", Context.MODE_PRIVATE)
        prefs.edit().putString("key", "value").apply()

        assertEquals("value", prefs.getString("key", null))
    }
}

@RunWith(RobolectricTestRunner::class)
class ResourceTest {
    private val context = ApplicationProvider.getApplicationContext<Context>()

    @Test
    fun stringResource_returnsCorrectValue() {
        val appName = context.getString(R.string.app_name)
        assertNotNull(appName)
    }
}
```

---

## Compose + Robolectric

```kotlin
@RunWith(RobolectricTestRunner::class)
class ComposeRobolectricTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun counter_incrementsOnClick() {
        composeTestRule.setContent {
            CounterScreen()
        }

        composeTestRule.onNodeWithText("0").assertIsDisplayed()
        composeTestRule.onNodeWithText("+1").performClick()
        composeTestRule.onNodeWithText("1").assertIsDisplayed()
    }
}
```

---

## まとめ

| 特徴 | 内容 |
|------|------|
| 実行環境 | JVM（実機不要） |
| 速度 | エミュレータの10倍以上 |
| Android API | シャドウクラスで模倣 |
| CI/CD | 高速フィードバック |

- Robolectricでエミュレータ不要の高速テスト
- `ApplicationProvider.getApplicationContext`でContext取得
- Compose UIテストもJVM上で実行可能
- CI/CDパイプラインでの高速テスト実行に最適

---

8種類のAndroidアプリテンプレート（テスト対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Espresso](https://zenn.dev/myougatheaxo/articles/android-compose-espresso-2026)
- [Compose Testing](https://zenn.dev/myougatheaxo/articles/android-compose-compose-testing-2026)
- [ViewModel Testing](https://zenn.dev/myougatheaxo/articles/android-compose-viewmodel-testing-2026)
