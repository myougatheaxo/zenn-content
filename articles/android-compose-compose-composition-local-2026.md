---
title: "CompositionLocal完全ガイド — LocalContext/LocalDensity/カスタムLocal"
emoji: "📍"
type: "tech"
topics: ["android", "jetpackcompose", "kotlin", "compositionlocal"]
published: true
---

## この記事で学べること

**CompositionLocal**（組み込みLocal、LocalContext、LocalDensity、カスタムLocal定義）を解説します。

---

## 組み込みCompositionLocal

```kotlin
@Composable
fun BuiltInLocals() {
    // Context取得
    val context = LocalContext.current

    // Density取得（dp⇔px変換）
    val density = LocalDensity.current
    val px = with(density) { 16.dp.toPx() }

    // ライフサイクルオーナー
    val lifecycleOwner = LocalLifecycleOwner.current

    // コンテンツカラー
    val contentColor = LocalContentColor.current

    // ハプティクス
    val haptic = LocalHapticFeedback.current

    // クリップボードマネージャ
    val clipboard = LocalClipboardManager.current

    Column(Modifier.padding(16.dp)) {
        Text("Density: ${density.density}")
        Text("16dp = ${px}px")
        Text("Content color: $contentColor")
    }
}
```

---

## 実用例

```kotlin
@Composable
fun DensityAwareLayout() {
    val density = LocalDensity.current
    val screenWidthDp = with(density) {
        LocalConfiguration.current.screenWidthDp.dp
    }

    if (screenWidthDp > 600.dp) {
        Row { /* タブレットレイアウト */ }
    } else {
        Column { /* スマホレイアウト */ }
    }
}

// Toast表示
@Composable
fun ToastExample() {
    val context = LocalContext.current

    Button(onClick = {
        Toast.makeText(context, "保存しました", Toast.LENGTH_SHORT).show()
    }) { Text("保存") }
}

// dp⇔px変換ユーティリティ
@Composable
fun CanvasWithDensity() {
    val density = LocalDensity.current

    Canvas(Modifier.size(100.dp)) {
        val radiusPx = with(density) { 20.dp.toPx() }
        drawCircle(Color.Blue, radius = radiusPx)
    }
}
```

---

## まとめ

| Local | 提供する値 |
|-------|----------|
| `LocalContext` | Android Context |
| `LocalDensity` | Density（dp⇔px） |
| `LocalLifecycleOwner` | LifecycleOwner |
| `LocalContentColor` | コンテンツカラー |
| `LocalHapticFeedback` | ハプティクス |
| `LocalClipboardManager` | クリップボード |

- 組み込みLocalでAndroidの各種サービスにアクセス
- `LocalDensity`でdp⇔px変換
- `LocalContext`でToast/Intent等のAndroid API
- カスタムLocalでアプリ固有の値を伝搬

---

8種類のAndroidアプリテンプレート（Compose対応）を公開しています。

**テンプレート一覧** → [Gumroad](https://myougatheax.gumroad.com)

関連記事：
- [CompositionLocalProvider](https://zenn.dev/myougatheaxo/articles/android-compose-compose-local-provider-2026)
- [Side Effect](https://zenn.dev/myougatheaxo/articles/android-compose-compose-side-effect-2026)
- [State Hoisting](https://zenn.dev/myougatheaxo/articles/android-compose-compose-state-hoisting-2026)
