---
title: "バージョン管理完全ガイド — セマンティックバージョニング/versionCode自動化"
emoji: "🏷️"
type: "tech"
topics: ["android", "kotlin", "gradle", "cicd"]
published: true
---

## この記事で学べること

**バージョン管理**（セマンティックバージョニング、versionCode自動化、Git連携、CI/CD統合、リリースノート）を解説します。

---

## セマンティックバージョニング

```
MAJOR.MINOR.PATCH
  ↓      ↓     ↓
破壊的   新機能  バグ修正
変更
```

```kotlin
// build.gradle.kts
android {
    defaultConfig {
        versionCode = 10203  // 1.2.3 → 10203
        versionName = "1.2.3"
    }
}
```

---

## Git Tag連携の自動バージョニング

```kotlin
// build.gradle.kts
fun getVersionName(): String {
    return try {
        val process = Runtime.getRuntime().exec("git describe --tags --abbrev=0")
        process.inputStream.bufferedReader().readText().trim().removePrefix("v")
    } catch (e: Exception) {
        "1.0.0"
    }
}

fun getVersionCode(): Int {
    return try {
        val process = Runtime.getRuntime().exec("git rev-list --count HEAD")
        process.inputStream.bufferedReader().readText().trim().toInt()
    } catch (e: Exception) {
        1
    }
}

android {
    defaultConfig {
        versionCode = getVersionCode()
        versionName = getVersionName()
    }
}
```

---

## version.propertiesファイル方式

```properties
# version.properties
VERSION_MAJOR=1
VERSION_MINOR=2
VERSION_PATCH=3
```

```kotlin
// build.gradle.kts
val versionProps = Properties().apply {
    file("version.properties").inputStream().use { load(it) }
}

val major = versionProps["VERSION_MAJOR"].toString().toInt()
val minor = versionProps["VERSION_MINOR"].toString().toInt()
val patch = versionProps["VERSION_PATCH"].toString().toInt()

android {
    defaultConfig {
        versionCode = major * 10000 + minor * 100 + patch
        versionName = "$major.$minor.$patch"
    }
}

// バージョンバンプタスク
tasks.register("bumpPatch") {
    doLast {
        val props = Properties().apply {
            file("version.properties").inputStream().use { load(it) }
        }
        props["VERSION_PATCH"] = (props["VERSION_PATCH"].toString().toInt() + 1).toString()
        file("version.properties").outputStream().use { props.store(it, null) }
    }
}
```

---

## Composeでバージョン表示

```kotlin
@Composable
fun AboutScreen() {
    val context = LocalContext.current
    val versionName = remember {
        context.packageManager.getPackageInfo(context.packageName, 0).versionName
    }
    val versionCode = remember {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            context.packageManager.getPackageInfo(context.packageName, 0).longVersionCode
        } else {
            @Suppress("DEPRECATION")
            context.packageManager.getPackageInfo(context.packageName, 0).versionCode.toLong()
        }
    }

    Column(Modifier.padding(16.dp)) {
        Text("バージョン情報", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))
        ListItem(
            headlineContent = { Text("バージョン") },
            supportingContent = { Text("$versionName ($versionCode)") }
        )
    }
}
```

---

## まとめ

| 方式 | 特徴 |
|------|------|
| Git Tag | リポジトリと自動同期 |
| version.properties | 明示的管理 |
| CI自動化 | ビルド番号自動インクリメント |
| BuildConfig | アプリ内でバージョン参照 |

- セマンティックバージョニングで意味のあるバージョン管理
- Git Tag連携で手動管理を排除
- CI/CDでversionCode自動インクリメント
- `PackageManager`でアプリ内にバージョン表示

---

8種類のAndroidアプリテンプレート（バージョン管理設定済み）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [GitHub Actions CI](https://zenn.dev/myougatheaxo/articles/android-compose-github-actions-ci-2026)
- [リリースチェックリスト](https://zenn.dev/myougatheaxo/articles/android-app-release-checklist-2026)
- [App Bundle](https://zenn.dev/myougatheaxo/articles/android-compose-app-bundle-delivery-2026)
