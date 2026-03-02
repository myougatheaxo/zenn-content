---
title: "Compose Desktop完全ガイド — マルチプラットフォーム/ウィンドウ管理/メニューバー"
emoji: "🖥️"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "kmp"]
published: true
---

## この記事で学べること

**Compose Desktop**（Compose Multiplatform、ウィンドウ管理、メニューバー、トレイアイコン）を解説します。

---

## 基本構造

```kotlin
fun main() = application {
    Window(
        onCloseRequest = ::exitApplication,
        title = "My App",
        state = rememberWindowState(
            width = 800.dp,
            height = 600.dp,
            position = WindowPosition(Alignment.Center)
        )
    ) {
        MaterialTheme {
            App()
        }
    }
}

@Composable
fun App() {
    var count by remember { mutableIntStateOf(0) }

    Column(
        Modifier.fillMaxSize().padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("Count: $count", style = MaterialTheme.typography.headlineLarge)
        Spacer(Modifier.height(16.dp))
        Button(onClick = { count++ }) { Text("Increment") }
    }
}
```

---

## メニューバー

```kotlin
fun main() = application {
    Window(onCloseRequest = ::exitApplication, title = "Editor") {
        MenuBar {
            Menu("ファイル") {
                Item("新規", onClick = { /* 新規作成 */ }, shortcut = KeyShortcut(Key.N, ctrl = true))
                Item("開く", onClick = { /* ファイルを開く */ }, shortcut = KeyShortcut(Key.O, ctrl = true))
                Item("保存", onClick = { /* 保存 */ }, shortcut = KeyShortcut(Key.S, ctrl = true))
                Separator()
                Item("終了", onClick = ::exitApplication)
            }
            Menu("編集") {
                Item("元に戻す", onClick = {}, shortcut = KeyShortcut(Key.Z, ctrl = true))
                Item("やり直し", onClick = {}, shortcut = KeyShortcut(Key.Z, ctrl = true, shift = true))
            }
        }

        MaterialTheme { EditorContent() }
    }
}
```

---

## 共有コード (expect/actual)

```kotlin
// commonMain
expect fun getPlatformName(): String

@Composable
fun SharedScreen() {
    Column {
        Text("プラットフォーム: ${getPlatformName()}")
        SharedButton()
    }
}

// androidMain
actual fun getPlatformName(): String = "Android"

// desktopMain
actual fun getPlatformName(): String = "Desktop"
```

---

## まとめ

| 機能 | API |
|------|-----|
| ウィンドウ | `Window` composable |
| メニュー | `MenuBar` |
| トレイ | `Tray` |
| ショートカット | `KeyShortcut` |

- Compose MultiplatformでAndroid/Desktop共通UI
- `MenuBar`でネイティブメニューバー
- `expect/actual`でプラットフォーム固有処理
- Kotlin共通コードで開発効率を最大化

---

8種類のAndroidアプリテンプレート（Compose対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [KMP共通コード](https://zenn.dev/myougatheaxo/articles/android-compose-kmp-shared-logic-2026)
- [Material3テーマ](https://zenn.dev/myougatheaxo/articles/android-compose-material3-theme-2026)
- [State管理](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-management-2026)
