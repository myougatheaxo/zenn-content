---
title: "Baseline Profile完全ガイド — AOTコンパイル/起動高速化/プロファイル生成"
emoji: "🚀"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "performance"]
published: true
---

## この記事で学べること

**Baseline Profile**（AOTコンパイル、プロファイル生成、起動高速化、CI連携）を解説します。

---

## プロファイル生成

```kotlin
// :baselineprofile モジュール
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() = rule.collect(
        packageName = "com.example.app"
    ) {
        // コールドスタート
        pressHome()
        startActivityAndWait()

        // メイン画面のスクロール
        device.findObject(By.res("item_list"))?.apply {
            fling(Direction.DOWN)
            device.waitForIdle()
            fling(Direction.UP)
            device.waitForIdle()
        }

        // 詳細画面遷移
        device.findObject(By.res("item_0"))?.click()
        device.waitForIdle()

        // 設定画面
        device.findObject(By.res("settings_button"))?.click()
        device.waitForIdle()
    }
}
```

---

## build.gradle設定

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("androidx.baselineprofile")
}

android {
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}

dependencies {
    baselineProfile(project(":baselineprofile"))
}

baselineProfile {
    automaticGenerationDuringBuild = true
}

// baselineprofile/build.gradle.kts
plugins {
    id("com.android.test")
    id("androidx.baselineprofile")
}

android {
    targetProjectPath = ":app"
}
```

---

## Startup Profile

```kotlin
@RunWith(AndroidJUnit4::class)
class StartupProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateStartupProfile() = rule.collect(
        packageName = "com.example.app",
        includeInStartupProfile = true
    ) {
        startActivityAndWait()
    }
}
```

---

## まとめ

| 機能 | 効果 |
|------|------|
| Baseline Profile | 初回起動高速化 |
| Startup Profile | 起動パス最適化 |
| AOTコンパイル | JITを回避 |
| `automaticGeneration` | ビルド時自動生成 |

- Baseline Profileで初回起動を30-50%高速化
- クリティカルパスのAOTコンパイルで最適化
- Startup Profileで起動に必要なクラスを事前コンパイル
- CI/CDで自動生成・更新

---

8種類のAndroidアプリテンプレート（パフォーマンス最適化済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [Benchmark](https://zenn.dev/myougatheaxo/articles/android-compose-compose-benchmark-2026)
- [Stable Annotation](https://zenn.dev/myougatheaxo/articles/android-compose-compose-stable-annotation-2026)
- [SplashScreen](https://zenn.dev/myougatheaxo/articles/android-compose-compose-splash-screen-2026)
